# Doc 101: QEMU UI/Display 子系统全面分析

## 文档信息
- **组件**: UI/Display (VNC, GTK, Spice, 输入子系统, 渲染管线)
- **源码版本**: QEMU 11.0.50
- **分析日期**: 2025-07
- **关联文档**: 无前置依赖（首篇 UI 分析）

---

## 目录
1. [架构概览](#1-架构概览)
2. [核心数据结构](#2-核心数据结构)
3. [Display 更新管线](#3-display-更新管线)
4. [DisplayChangeListener 观察者模式](#4-displaychangelistener-观察者模式)
5. [Surface 与 Pixman](#5-surface-与-pixman)
6. [VNC 实现](#6-vnc-实现)
7. [GTK UI](#7-gtk-ui)
8. [Spice 显示](#8-spice-显示)
9. [OpenGL/EGL 与 dmabuf 零拷贝](#9-openglegl-与-dmabuf-零拷贝)
10. [输入子系统](#10-输入子系统)
11. [线程模型](#11-线程模型)
12. [构建配置](#12-构建配置)
13. [多后端并存机制](#13-多后端并存机制)
14. [性能优化要点](#14-性能优化要点)
15. [总结](#15-总结)

---

## 1. 架构概览

QEMU 的显示子系统采用 **观察者模式（Observer Pattern）**，将 GPU/显示设备的帧缓冲输出与多个显示后端解耦：

```
┌─────────────────────────────────────────────────────────────────┐
│                      GPU / 显示设备                               │
│  (VGA, virtio-gpu, QXL, ramfb, bochs-display, ...)             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ graphic_hw_update()
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    QemuConsole                                    │
│  (QemuGraphicConsole / QemuTextConsole)                         │
│  持有: DisplaySurface, dirty bitmap                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │ dpy_gfx_update() / dpy_gfx_switch()
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DisplayState                                   │
│  管理所有 DisplayChangeListener (链表)                           │
│  定时器驱动 dpy_refresh()                                        │
└───────┬──────────────┬──────────────┬───────────────────────────┘
        │              │              │
        ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐   ┌──────────┐
   │   VNC   │   │   GTK   │   │  Spice   │
   │ Listener│   │ Listener│   │ Listener │
   └─────────┘   └─────────┘   └──────────┘
```

**关键设计原则：**
- GPU 设备不知道也不关心有哪些显示后端
- 多个后端可同时活跃（如 VNC + GTK 同时输出）
- 每个后端独立消费 dirty 矩形和 surface 切换事件
- Pixman 作为统一的 CPU 侧帧缓冲抽象

---

## 2. 核心数据结构

### 2.1 DisplayState — 全局显示注册表

```c
// ui/console.c:62-69
struct DisplayState {
    QEMUTimer *gui_timer;           // 刷新定时器
    QLIST_HEAD(, DisplayChangeListener) listeners;  // 后端链表
    // ...
};
```

全局唯一实例，管理所有注册的显示后端和刷新节奏。

### 2.2 QemuConsole — 每输出一个 Console

```c
// ui/console.c:349-416 (QemuGraphicConsole 继承)
struct QemuConsole {
    Object parent;
    int index;                      // console 编号
    DisplaySurface *surface;        // 当前帧缓冲
    // ...
};

struct QemuGraphicConsole {
    QemuConsole parent;
    Object *device;                 // 关联的 GPU 设备
    const GraphicHwOps *hw_ops;     // GPU 回调 (update/invalidate/text_update)
    void *hw;                       // GPU 设备 opaque
    // ...
};
```

- 每个 GPU 设备创建一个或多个 `QemuGraphicConsole`
- `hw_ops->gfx_update()` 是从 GPU 拉取最新帧的入口

### 2.3 DisplaySurface — 帧缓冲封装

```c
// ui/display-surface.c:24-107
struct DisplaySurface {
    pixman_image_t *image;          // pixman 图像 (实际像素数据)
    pixman_format_code_t format;    // 像素格式
    int width, height;
    int linesize;                   // 每行字节数（含对齐）
    // flags: shared, dmabuf, GL texture, ...
};
```

### 2.4 DisplayChangeListener — 后端接口

```c
// include/ui/console.h:193-265
struct DisplayChangeListenerOps {
    const char *dpy_name;

    /* 核心回调 */
    void (*dpy_refresh)(DisplayChangeListener *dcl);
    void (*dpy_gfx_update)(DisplayChangeListener *dcl, int x, int y, int w, int h);
    void (*dpy_gfx_switch)(DisplayChangeListener *dcl, struct DisplaySurface *new_surface);

    /* 光标 */
    void (*dpy_cursor_define)(DisplayChangeListener *dcl, QEMUCursor *cursor);
    void (*dpy_mouse_set)(DisplayChangeListener *dcl, int x, int y, int on);

    /* GL 路径 */
    void (*dpy_gl_scanout_disable)(DisplayChangeListener *dcl);
    void (*dpy_gl_scanout_texture)(DisplayChangeListener *dcl, ...);
    void (*dpy_gl_scanout_dmabuf)(DisplayChangeListener *dcl, QemuDmaBuf *dmabuf);
    void (*dpy_gl_update)(DisplayChangeListener *dcl, ...);
    void (*dpy_gl_cursor_dmabuf)(DisplayChangeListener *dcl, ...);

    /* 窗口管理 */
    bool (*dpy_has_dmabuf)(DisplayChangeListener *dcl);
    // ...
};

struct DisplayChangeListener {
    uint64_t update_interval;       // 刷新间隔（纳秒）
    const DisplayChangeListenerOps *ops;
    DisplayState *ds;
    QemuConsole *con;               // 绑定的 console
    QLIST_ENTRY(DisplayChangeListener) next;
};
```

---

## 3. Display 更新管线

### 3.1 完整数据流

```
时间线:
────────────────────────────────────────────────────────────────────
  [定时器触发]          [GPU刷新]            [后端消费]
       │                    │                    │
       ▼                    │                    │
  dpy_refresh()             │                    │
       │                    │                    │
       ├─► listener.refresh()                    │
       │         │          │                    │
       │         ▼          │                    │
       │   graphic_hw_update()                   │
       │         │          │                    │
       │         ▼          │                    │
       │   hw_ops->gfx_update()                  │
       │         │          │                    │
       │         │   GPU写回surface              │
       │         │   + 标记 dirty rect           │
       │         │          │                    │
       │         ▼          ▼                    │
       │   dpy_gfx_update(con, x,y,w,h)         │
       │         │                               │
       │         ├──────► VNC: vnc_dpy_update()  │
       │         ├──────► GTK: gd_update()       │
       │         └──────► Spice: display_update()│
       │                                         │
────────────────────────────────────────────────────────────────────
```

### 3.2 关键函数

```c
// ui/console.c:137-145
void graphic_hw_update(QemuConsole *con)
{
    QemuGraphicConsole *gcon = QEMU_GRAPHIC_CONSOLE(con);
    if (gcon->hw_ops->gfx_update) {
        gcon->hw_ops->gfx_update(gcon->hw);
    }
}

// ui/console.c:75-107
static void dpy_refresh(DisplayState *s)
{
    DisplayChangeListener *dcl;
    QLIST_FOREACH(dcl, &s->listeners, next) {
        if (dcl->ops->dpy_refresh) {
            dcl->ops->dpy_refresh(dcl);
        }
    }
}

// ui/console.c:573-611
void dpy_gfx_update(QemuConsole *con, int x, int y, int w, int h)
{
    DisplayChangeListener *dcl;
    // 广播给所有绑定到该 console 的 listener
    QLIST_FOREACH(dcl, &con->listeners, next) {
        if (dcl->ops->dpy_gfx_update) {
            dcl->ops->dpy_gfx_update(dcl, x, y, w, h);
        }
    }
}
```

### 3.3 刷新频率控制

- `gui_timer` 默认 30ms 间隔（~33 FPS）
- 各后端可设置 `update_interval` 请求不同频率
- 系统取所有后端中最短间隔作为实际定时器周期

---

## 4. DisplayChangeListener 观察者模式

### 4.1 注册流程

```c
// 后端初始化时注册:
void register_displaychangelistener(DisplayChangeListener *dcl)
{
    // 加入 DisplayState 链表
    QLIST_INSERT_HEAD(&display_state->listeners, dcl, next);
    // 绑定到指定 console
    // 触发初始 dpy_gfx_switch() 使后端获取当前 surface
}
```

### 4.2 各后端注册示例

| 后端 | ops 变量 | 注册位置 |
|------|----------|----------|
| VNC | `dcl_ops` | `vnc_display_init()` → `ui/vnc.c` |
| GTK (软件) | `dcl_ops` | `gtk_display_init()` → `ui/gtk.c:555` |
| GTK (GL Area) | `dcl_gl_area_ops` | `ui/gtk.c:580` |
| GTK (EGL) | `dcl_egl_ops` | `ui/gtk.c:620` |
| Spice | `display_listener_ops` | `ui/spice-display.c:372` |
| EGL-headless | 专用 ops | `ui/egl-headless.c` |

### 4.3 Surface 切换

当 GPU 分辨率/格式变化时：
1. GPU 调用 `dpy_gfx_replace_surface(con, new_surface)`
2. 框架广播 `dpy_gfx_switch()` 给所有 listener
3. 各后端重新分配内部缓冲（VNC 重分配 server surface, GTK 重建 pixbuf）

---

## 5. Surface 与 Pixman

### 5.1 Pixman 的角色

Pixman 是 QEMU 显示子系统的基石库：

| 用途 | API |
|------|-----|
| 创建帧缓冲 | `pixman_image_create_bits()` |
| 像素格式转换 | `pixman_image_composite()` |
| 区域裁剪/拷贝 | `pixman_blt()` |
| 文本渲染（placeholder） | glyph helpers |
| 颜色空间管理 | `pixman_format_code_t` |

### 5.2 Surface 创建

```c
// ui/display-surface.c:35-61
DisplaySurface *qemu_create_displaysurface_from(int width, int height,
                                                 pixman_format_code_t format,
                                                 int linesize, uint8_t *data)
{
    DisplaySurface *surface = g_new0(DisplaySurface, 1);
    surface->format = format;
    surface->image = pixman_image_create_bits(format, width, height,
                                              (void *)data, linesize);
    return surface;
}
```

### 5.3 Placeholder Surface

当 console 没有活跃 GPU 时，创建带文字提示的占位 surface：

```c
// ui/display-surface.c:75-97
DisplaySurface *qemu_create_placeholder_surface(int w, int h, const char *msg)
{
    // 创建黑色背景 surface
    // 用 pixman glyph 渲染提示文字（如 "Display not active"）
}
```

---

## 6. VNC 实现

### 6.1 架构概览

```
┌──────────────────────────────────────────────┐
│              VncDisplay (全局)                 │
│  - listening sockets                          │
│  - 认证配置 (none/vnc/vencrypt/sasl)          │
│  - server_surface (共享帧缓冲副本)            │
│  - clients[] (VncState 链表)                  │
└──────────────────┬───────────────────────────┘
                   │ 每个连接
                   ▼
┌──────────────────────────────────────────────┐
│              VncState (每客户端)               │
│  - dirty bitmap (脏矩形追踪)                  │
│  - encoding (Raw/Hextile/Tight/ZRLE)         │
│  - I/O buffers (input/output/jobs_buffer)    │
│  - zlib/tight compression state              │
│  - TLS/SASL session state                    │
└──────────────────────────────────────────────┘
```

### 6.2 核心数据结构

```c
// ui/vnc.h:143-189
struct VncDisplay {
    QTAILQ_HEAD(, VncState) clients;    // 连接的客户端
    QIONetListener *listener;            // 监听 socket
    DisplaySurface *ds;                  // 当前 guest surface
    VncSurface server;                   // 服务端缓冲（编码源）
    // 认证:
    VncServerInfo *auth;
    int auth_method;                     // VNC_AUTH_NONE/VNC/VENCRYPT/SASL
    // TLS:
    QCryptoTLSCreds *tlscreds;
    // ...
};

// ui/vnc.h:265-351
struct VncState {
    VncDisplay *vd;
    QIOChannel *ioc;                     // I/O 通道
    Buffer input, output;                // 协议 I/O 缓冲
    Buffer jobs_buffer;                  // 编码线程输出缓冲
    VncStateBit dirty[VNC_MAX_HEIGHT];   // 脏矩形位图
    int encoding;                        // 当前编码
    VncTight tight;                      // Tight 压缩状态
    z_stream zlib_stream[4];             // Zlib 压缩流
    // ...
};
```

### 6.3 连接与认证流程

```
客户端连接 → vnc_connect() → VncState 创建
    → vnc_start_protocol() → RFB 版本协商
    → protocol_client_auth() → 认证分支:
        ├── VNC_AUTH_NONE → 直接通过
        ├── VNC_AUTH_VNC → start_auth_vnc()
        │       → 发送 challenge → 客户端 DES 加密回应
        │       → protocol_client_auth_vnc() 校验
        ├── VNC_AUTH_VENCRYPT → start_auth_vencrypt()
        │       → TLS 握手 → 子认证
        └── VNC_AUTH_SASL → start_auth_sasl()
                → SASL 机制协商 → step 交互
    → vnc_client_setup_encodings()
    → 进入正常帧更新循环
```

### 6.4 帧缓冲更新管线

```
1. dpy_gfx_update() 触发
       ↓
2. vnc_dpy_update(dcl, x, y, w, h)     [ui/vnc.c:669-676]
   → 标记 VncSurface.dirty 位图中对应区域
       ↓
3. vnc_update_server_surface()          [ui/vnc.c:793-814]
   → 将 guest surface 差异拷贝到 server surface
   → server surface 是编码的数据源（避免 guest 修改时竞争）
       ↓
4. vnc_update_client(vs)                [ui/vnc.c:1149-1217]
   → 遍历 VncState.dirty
   → 对每个脏矩形创建 VncJob
       ↓
5. vnc_send_framebuffer_update()        [ui/vnc.c:955-985]
   → 根据 vs->encoding 分发:
     ├── VNC_ENCODING_RAW → vnc_raw_send_framebuffer_update()
     ├── VNC_ENCODING_HEXTILE → vnc_hextile_send_framebuffer_update()
     ├── VNC_ENCODING_TIGHT → vnc_tight_send_framebuffer_update()
     ├── VNC_ENCODING_ZLIB → vnc_zlib_send_framebuffer_update()
     └── VNC_ENCODING_ZRLE → vnc_zrle_send_framebuffer_update()
       ↓
6. 编码结果写入 jobs_buffer
       ↓
7. vnc_jobs_consume_buffer()            [主线程消费]
   → 将 jobs_buffer 追加到 output buffer
   → 通过 QIOChannel 发送给客户端
```

### 6.5 编码算法对比

| 编码 | 特点 | 适用场景 |
|------|------|----------|
| **Raw** | 无压缩, 带宽最大 | 本地/高带宽局域网 |
| **Hextile** | 16×16 分块, 简单 RLE | 低 CPU, 中等带宽 |
| **Tight** | JPEG + zlib 混合 | 照片/视频类内容 |
| **Zlib** | 整帧 zlib 压缩 | 文本/桌面场景 |
| **ZRLE** | Run-Length + zlib, 分块 | 综合最优性能 |

### 6.6 VNC Worker 线程

```c
// ui/vnc-jobs.c 线程模型
struct VncJobQueue {
    QemuMutex mutex;
    QemuCond cond;
    QTAILQ_HEAD(, VncJob) jobs;     // 待处理 job 队列
    bool exit;
};

// 工作流:
// 主线程: vnc_job_push(job) → signal cond
// Worker: vnc_worker_thread_loop() → 取 job → 编码 → 写入 vs->jobs_buffer
// 主线程: vnc_jobs_consume_buffer(vs) → 取 jobs_buffer → 追加到 output → 网络发送
```

**锁分层** (ui/vnc.c:36-53 注释文档):
- `VncDisplay.mutex` — 保护客户端链表
- `VncState.output_mutex` — 保护 output/jobs_buffer
- `VncJobQueue.mutex` — 保护 job 队列

---

## 7. GTK UI

### 7.1 架构

GTK 后端是本地窗口显示，直接将帧缓冲绘制到 GTK 窗口：

```c
// ui/gtk.c 核心结构
struct GtkDisplayState {
    GtkWidget *window;              // 主窗口
    GtkWidget *menu_bar;            // 菜单栏
    VirtualConsole vc[MAX_VCS];     // 多 console 标签页
    // ...
};

struct VirtualConsole {
    GtkDisplayState *s;
    QemuConsole *con;               // 绑定的 QEMU console
    GtkWidget *drawing_area;        // 绘图区域
    DisplayChangeListener dcl;      // listener 实例
    DisplaySurface *ds;             // 当前 surface 引用
    // ...
};
```

### 7.2 三种渲染模式

| 模式 | ops | 条件 | 特点 |
|------|-----|------|------|
| 软件渲染 | `dcl_ops` | 默认 | pixman → cairo → GDK |
| GL Area | `dcl_gl_area_ops` | GTK GL Widget 可用 | OpenGL 纹理路径 |
| EGL/X11 | `dcl_egl_ops` | X11 + EGL 可用 | 外部 EGL surface |

### 7.3 软件渲染路径

```c
// ui/gtk.c:386-435
static void gd_update(DisplayChangeListener *dcl, int x, int y, int w, int h)
{
    VirtualConsole *vc = container_of(dcl, VirtualConsole, dcl);
    // 1. 从 DisplaySurface 的 pixman image 获取像素数据
    // 2. 通过 cairo_surface 创建 GDK 可绘制对象
    // 3. 标记 drawing_area 的脏区域
    // 4. GTK 事件循环中触发重绘 → cairo_paint()
    gtk_widget_queue_draw_area(vc->drawing_area, x, y, w, h);
}

// ui/gtk.c:437-440
static void gd_refresh(DisplayChangeListener *dcl)
{
    graphic_hw_update(dcl->con);    // 拉取 GPU 最新帧
}

// ui/gtk.c:492-553
static void gd_switch(DisplayChangeListener *dcl, DisplaySurface *surface)
{
    // 分辨率/格式变化 → 重建 pixman → cairo → resize 窗口
}
```

### 7.4 输入事件处理

```c
// ui/gtk.c:980-1212
// 鼠标移动
static gboolean gd_motion_event(GtkWidget *w, GdkEventMotion *m, void *opaque)
{
    // 坐标转换 (窗口坐标 → guest 坐标)
    // → qemu_input_queue_abs() 或 qemu_input_queue_rel()
    // → qemu_input_event_sync()
}

// 按键
static gboolean gd_key_event(GtkWidget *w, GdkEventKey *k, void *opaque)
{
    // GDK keyval → QKeyCode 映射
    // → qemu_input_event_send_key_qcode()
}

// 触摸 (multi-touch)
static gboolean gd_touch_event(GtkWidget *w, GdkEventTouch *t, void *opaque)
{
    // → qemu_input_touch_event()
}
```

---

## 8. Spice 显示

### 8.1 架构差异

Spice 与 VNC 有本质架构区别：

| 维度 | VNC | Spice |
|------|-----|-------|
| 协议模型 | 单一 RFB 流 | 多通道 (main/display/input/cursor/...) |
| 编码位置 | QEMU 进程内 | SPICE Server 库内 |
| 设备集成 | 通用 DisplayChangeListener | 专用 QXL 设备接口 |
| 渲染命令 | 位图更新 | 高级绘图命令 (copy/fill/draw) |
| 客户端智能 | 哑终端 | 支持客户端渲染/缓存 |
| 音视频 | 不支持 | 原生支持 |

### 8.2 核心组件

```
┌───────────────────────────────────────────────────┐
│  QEMU 进程                                         │
│  ┌─────────────────┐    ┌──────────────────────┐  │
│  │  QXL Device     │    │  SimpleSpiceDisplay  │  │
│  │  (hw/display/)  │    │  (ui/spice-display.c)│  │
│  │                 │◄──►│                      │  │
│  │  QXLInterface   │    │  DisplayListener     │  │
│  └────────┬────────┘    └──────────┬───────────┘  │
│           │                        │               │
│           ▼                        ▼               │
│  ┌─────────────────────────────────────────────┐  │
│  │          SpiceServer (libspice)              │  │
│  │  - Display channel                          │  │
│  │  - Input channel                            │  │
│  │  - Cursor channel                           │  │
│  │  - 编码/压缩/传输                            │  │
│  └──────────────────────┬──────────────────────┘  │
└─────────────────────────┼─────────────────────────┘
                          │ TCP/Unix Socket
                          ▼
                   ┌──────────────┐
                   │ Spice Client │
                   └──────────────┘
```

### 8.3 Spice Core 初始化

```c
// ui/spice-core.c:662-760
void qemu_spice_init(void)
{
    // 1. 创建 SpiceServer 实例
    // 2. 配置 TLS/SASL 认证
    // 3. 注册 timer/watch 回调到 QEMU 主循环
    // 4. 设置端口和迁移参数
    // 5. spice_server_init() 启动服务
}
```

### 8.4 显示更新路径

```c
// ui/spice-display.c:124-256
// Spice 通过 QXL 接口提交绘图命令:

static void qemu_spice_create_update(SimpleSpiceDisplay *ssd)
{
    // 1. 从 dirty 区域生成 QXLDrawable (DRAW_COPY 命令)
    // 2. 附带源数据: 从 surface pixman image 拷贝脏区域
    // 3. 提交到 SpiceServer 的 display channel
}

// ui/spice-display.c:374-459
static const DisplayChangeListenerOps display_listener_ops = {
    .dpy_name          = "spice",
    .dpy_gfx_update    = display_update,      // 标记 dirty
    .dpy_gfx_switch    = display_switch,      // surface 切换
    .dpy_refresh       = display_refresh,     // 生成 QXL 命令
    // GL 路径:
    .dpy_gl_scanout_dmabuf = spice_gl_scanout_dmabuf,
    .dpy_gl_update     = spice_gl_update,
};
```

### 8.5 QXL Interface

```c
// ui/spice-display.c:697-719
static const QXLInterface dod_interface = {
    .get_command       = interface_get_command,
    .req_cmd_notification = interface_req_cmd_notification,
    .release_resource  = interface_release_resource,
    .get_cursor_command = interface_get_cursor_command,
    // ...
};
```

SpiceServer 主动从 QEMU 拉取 QXL 命令（拉模式），而非 QEMU 推送。

---

## 9. OpenGL/EGL 与 dmabuf 零拷贝

### 9.1 GL 显示路径

对于 virtio-gpu 等支持 3D 加速的设备，显示数据以 GPU 纹理/dmabuf 形式传递：

```
virtio-gpu (guest 渲染)
    → virgl/venus (host GPU 渲染)
    → GL texture / dmabuf FD
    → DisplayChangeListenerOps.dpy_gl_scanout_dmabuf()
    → 后端直接导入纹理（零拷贝）
```

### 9.2 DisplayGLCtx

```c
// include/ui/console.h (GL 兼容性)
struct DisplayGLCtx {
    // GL 上下文创建/销毁回调
    // 确保后端 GL 上下文与 console GL 上下文兼容
};

// ui/console.c:267-290
bool dpy_gl_ctx_is_compatible_dcl(QemuConsole *con, DisplayChangeListener *dcl)
{
    // 检查 dcl 是否支持当前 console 的 GL 模式
}
```

### 9.3 dmabuf 零拷贝流程

```
1. GPU 设备生成 dmabuf FD (DMA-BUF 文件描述符)
2. 调用 dpy_gl_scanout_dmabuf(dcl, dmabuf)
3. 后端处理:
   - GTK: 通过 EGL import → GL texture → GTK GL area 绘制
   - Spice: 转发 dmabuf FD → SPICE GL scanout → 客户端 import
   - egl-headless: import → 可选 readback 到 pixman
4. 帧完成: dpy_gl_update() 通知后端呈现
```

### 9.4 EGL-Headless 后端

```c
// ui/egl-headless.c
// 无窗口 GL 渲染后端:
// - 创建 EGL 上下文（offscreen）
// - 接收 dmabuf/texture scanout
// - 可选: readback 到 pixman surface 供其他后端使用
// - 用途: 服务器无 X11 时仍可用 GPU 加速 + VNC 输出
```

### 9.5 udmabuf

```c
// ui/udmabuf.c (Linux 特有)
// 将 QEMU RAM 区域导出为 dmabuf:
// - 通过 /dev/udmabuf 设备
// - guest framebuffer → dmabuf FD → 后端零拷贝导入
// - 避免 guest RAM → CPU 拷贝 → 后端
```

---

## 10. 输入子系统

### 10.1 架构

```
┌─────────────────┐     ┌──────────────────────────┐
│  输入前端        │     │  输入后端（设备）          │
│  (GTK/VNC/Spice)│     │  (PS/2, USB HID, virtio) │
│                 │     │                          │
│  产生事件        │────►│  QemuInputHandler        │
└─────────────────┘     └──────────────────────────┘
         │                         ▲
         ▼                         │
┌─────────────────────────────────────────────────┐
│              Input Core (ui/input.c)             │
│  - handler 注册/绑定                             │
│  - 事件路由 (console 绑定优先)                    │
│  - 队列化/去抖                                   │
└─────────────────────────────────────────────────┘
```

### 10.2 核心接口

```c
// include/ui/input.h:25-30
struct QemuInputHandler {
    const char *name;
    uint32_t mask;                          // 支持的事件类型 bitmap
    void (*event)(DeviceState *dev, QemuConsole *src, InputEvent *evt);
    void (*sync)(DeviceState *dev);         // 事件批次完成
};
```

### 10.3 注册与路由

```c
// ui/input.c:48-99
QemuInputHandlerState *qemu_input_handler_register(DeviceState *dev,
                                                    const QemuInputHandler *handler)
{
    // 将 handler 插入全局有序链表
    // 优先级: console-bound > generic
}

// ui/input.c:100-123
static QemuInputHandlerState *qemu_input_find_handler(uint32_t mask,
                                                       QemuConsole *con)
{
    // 1. 先找绑定到指定 console 的 handler
    // 2. 如果没有，找支持该事件类型的通用 handler
    // 3. 返回第一个匹配的
}

// ui/input.c:306-319
void qemu_input_event_send_impl(QemuConsole *src, InputEvent *evt)
{
    QemuInputHandlerState *s = qemu_input_find_handler(evt_mask, src);
    s->handler->event(s->dev, src, evt);
}
```

### 10.4 事件类型

```c
// 键盘事件
qemu_input_event_send_key_number(con, keycode, down);
qemu_input_event_send_key_qcode(con, qcode, down);

// 鼠标事件
qemu_input_queue_rel(con, INPUT_AXIS_X, dx);   // 相对移动
qemu_input_queue_abs(con, INPUT_AXIS_X, x, 0, max);  // 绝对定位
qemu_input_queue_btn(con, btn, down);          // 按键

// 触摸事件
qemu_input_touch_event(con, slot, x, y, pressure, ...);

// 批次同步
qemu_input_event_sync();  // 通知设备: 一批事件发送完毕
```

### 10.5 输入流程示例 (VNC 鼠标)

```
VNC 客户端发送鼠标移动包
    → vnc_read_client() 解析 RFB 协议
    → pointer_event() [ui/vnc.c]
    → qemu_input_queue_abs(con, INPUT_AXIS_X, x, ...)
    → qemu_input_queue_abs(con, INPUT_AXIS_Y, y, ...)
    → qemu_input_event_sync()
    → 路由到绑定的 QemuInputHandler (如 USB tablet)
    → USB HID 设备更新状态 → guest 通过 USB 中断获取位置
```

---

## 11. 线程模型

### 11.1 总体设计

```
┌───────────────────────────────────────────────────────────────┐
│                      QEMU 主线程                               │
│  - 主事件循环 (glib main loop)                                 │
│  - gui_timer 触发 dpy_refresh()                               │
│  - VNC 协议处理 (socket I/O)                                   │
│  - GTK 事件处理 (窗口/输入)                                    │
│  - Spice Server 回调                                          │
│  - 输入事件路由                                                │
└───────────────────────────────┬───────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ VNC Worker   │  │ Spice Server │  │ vCPU 线程    │
    │ Thread(s)    │  │ Thread(s)    │  │              │
    │              │  │              │  │ GPU 设备调用  │
    │ 编码压缩     │  │ 编码/传输    │  │ gfx_update() │
    └──────────────┘  └──────────────┘  └──────────────┘
```

### 11.2 VNC 线程同步

```
主线程:
  1. dpy_refresh() → graphic_hw_update()
  2. vnc_update_client() → 创建 VncJob
  3. vnc_job_push() → 加入队列, signal cond
  4. [继续处理其他事件]
  5. vnc_jobs_consume_buffer() → 获取编码结果 → 网络发送

Worker 线程:
  1. vnc_worker_thread_loop() → wait cond
  2. 取 VncJob
  3. 编码 (hextile/tight/zrle)
  4. 写入 vs->jobs_buffer (output_mutex 保护)
  5. 回到等待
```

### 11.3 为什么 VNC 不完全异步？

- 网络发送仍在主线程（因为 QIOChannel 与主循环集成）
- 只有 CPU 密集型的编码操作被卸载到 worker
- 避免了复杂的跨线程 socket 管理

---

## 12. 构建配置

### 12.1 Meson 构建选项

```meson
// ui/meson.build 关键片段

# 核心（始终编译）
system_ss.add(files('console.c', 'display-surface.c', 'input.c', ...))

# VNC（可选, 需 pixman）
if vnc.found() and pixman.found()
  vnc_ss = ss.source_set()
  vnc_ss.add(files('vnc.c', 'vnc-enc-hextile.c', 'vnc-enc-tight.c',
                   'vnc-enc-zlib.c', 'vnc-enc-zrle.c', 'vnc-jobs.c'))
  # 可选 SASL/TLS:
  vnc_ss.add(when: sasl, if_true: files('vnc-auth-sasl.c'))
endif

# GTK（可选）
if gtk.found()
  gtk_ss = ss.source_set()
  gtk_ss.add(files('gtk.c', 'gtk-clipboard.c'))
  # GL 扩展:
  if opengl.found()
    gtk_ss.add(files('gtk-gl-area.c'))  # GL Area 路径
  endif
  if x11.found() and opengl.found()
    gtk_ss.add(files('gtk-egl.c'))      # EGL/X11 路径
  endif
endif

# Spice（可选）
if spice.found()
  spice_ss.add(files('spice-core.c', 'spice-input.c', 'spice-display.c'))
endif

# EGL Headless（可选, 需 OpenGL + pixman）
if opengl.found() and pixman.found()
  system_ss.add(files('egl-headless.c'))
endif
```

### 12.2 模块依赖关系

```
console.c (核心) ─── 必须
    │
    ├── VNC ─── 需: pixman, 可选: gnutls(TLS), cyrus-sasl
    │
    ├── GTK ─── 需: gtk3, 可选: opengl, x11
    │
    ├── Spice ─── 需: spice-server, spice-protocol
    │
    ├── SDL ─── 需: SDL2
    │
    └── EGL-headless ─── 需: EGL, OpenGL, pixman
```

---

## 13. 多后端并存机制

### 13.1 同时运行

QEMU 支持多个显示后端同时活跃：

```bash
# 典型配置: VNC + 本地 GTK 窗口
qemu-system-aarch64 \
    -display gtk \           # 本地窗口
    -vnc :0                  # VNC 远程访问

# 无头 + VNC (服务器场景)
qemu-system-aarch64 \
    -display none \          # 无本地窗口
    -vnc :0                  # 只有 VNC
```

### 13.2 多后端更新广播

```c
void dpy_gfx_update(QemuConsole *con, int x, int y, int w, int h)
{
    DisplayChangeListener *dcl;
    // 遍历所有注册在该 console 上的 listener
    QLIST_FOREACH(dcl, &con->listeners, next) {
        dcl->ops->dpy_gfx_update(dcl, x, y, w, h);
    }
    // VNC listener: 标记 dirty → 编码 → 网络
    // GTK listener: 标记 dirty → cairo repaint → 本地窗口
    // Spice listener: 标记 dirty → QXL command → SPICE server
}
```

### 13.3 Console 与 Listener 的关系

- 一个 Console 可有多个 Listener（多后端同时输出）
- 一个 Listener 绑定一个 Console（单 console 视图）
- VNC 是特殊情况：一个 VncDisplay 对应一个 Listener，但服务多个 VncState 客户端

---

## 14. 性能优化要点

### 14.1 脏矩形追踪

避免全屏重传：
- GPU 只标记实际修改的矩形区域
- 各后端只处理脏区域（非全帧）
- VNC 按 16×16 tile 粒度追踪

### 14.2 Server Surface 双缓冲（VNC）

```
Guest Surface (GPU 持续修改)
    │
    │ 拷贝脏区域
    ▼
Server Surface (编码线程安全读取)
    │
    │ 编码
    ▼
Wire (网络传输)
```

- 避免编码线程读取时 GPU 修改同一内存
- 只拷贝脏区域，非全帧

### 14.3 编码线程卸载

- 主线程不阻塞在编码上
- Tight (JPEG) 编码特别 CPU 密集
- Worker 线程并行处理多个客户端的编码

### 14.4 dmabuf 零拷贝路径

```
零拷贝: GPU 渲染 → dmabuf FD → 后端 import texture → 直接显示
传统:   GPU 渲染 → 回读到 RAM → pixman 转换 → 编码 → 网络
```

- 省去 GPU→CPU 回读
- 省去 pixman 格式转换
- 适用于 virtio-gpu + GTK/Spice GL 路径

### 14.5 自适应刷新率

```c
// 根据后端消费能力调整:
// - 快速后端（本地 GTK）: 60 FPS
// - 慢速后端（远程 VNC, 低带宽）: 降低到 10-30 FPS
// DisplayChangeListener.update_interval 控制
```

---

## 15. 总结

### 15.1 设计亮点

| 设计 | 实现 | 价值 |
|------|------|------|
| 观察者模式 | DisplayChangeListener 链表 | 后端完全解耦, 可热插拔 |
| 统一帧缓冲 | pixman image | 格式无关, 设备与后端独立 |
| 异步编码 | VNC worker threads | 主循环不阻塞 |
| 零拷贝 | dmabuf + GL scanout | 避免 CPU 拷贝, 降低延迟 |
| 多通道 | Spice channels | 音视频分离, 客户端智能 |
| 脏矩形 | dirty bitmap | 最小化更新量 |

### 15.2 数据流路径对比

| 路径 | 延迟 | 带宽 | CPU |
|------|------|------|-----|
| VNC Raw | 低 | 极高 | 低 |
| VNC Tight | 中 | 低 | 高 |
| GTK 软件 | 极低 | - | 中 |
| GTK GL | 极低 | - | 低 |
| Spice QXL | 中 | 中 | 中 |
| Spice GL | 低 | 低 | 低 |
| dmabuf 零拷贝 | 极低 | - | 极低 |

### 15.3 源文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `ui/console.c` | ~1800 | 核心: DisplayState, Console, update pipeline |
| `ui/display-surface.c` | ~110 | Surface 创建/管理 |
| `ui/vnc.c` | ~3500 | VNC 服务: 协议/认证/帧更新 |
| `ui/vnc.h` | ~450 | VNC 数据结构定义 |
| `ui/vnc-jobs.c` | ~260 | VNC 编码线程 |
| `ui/vnc-enc-tight.c` | ~1600 | Tight 编码 (JPEG+zlib) |
| `ui/vnc-enc-zrle.c` | ~400 | ZRLE 编码 |
| `ui/gtk.c` | ~2500 | GTK 窗口/输入/GL |
| `ui/spice-core.c` | ~800 | Spice 服务初始化/配置 |
| `ui/spice-display.c` | ~800 | Spice 显示桥接/QXL |
| `ui/input.c` | ~700 | 输入事件路由 |
| `ui/egl-headless.c` | ~200 | 无头 GL 渲染 |
| `ui/dmabuf.c` | ~150 | DMA-BUF 管理 |
| `include/ui/console.h` | ~500 | 核心接口定义 |

---

## 附录 A: 命令行配置参考

```bash
# VNC 配置
-vnc :0                          # 监听 5900 端口
-vnc :0,password=on              # 需要密码
-vnc :0,tls-creds=id            # TLS 加密
-vnc :0,sasl=on                  # SASL 认证

# GTK 配置
-display gtk                     # 基本 GTK 窗口
-display gtk,gl=on               # 启用 OpenGL
-display gtk,show-cursor=on      # 显示光标

# Spice 配置
-spice port=5930,disable-ticketing=on   # 基本 Spice
-spice gl=on                            # GL 加速
-display spice-app                      # Spice + vdagent (自动启动客户端)

# 无头模式
-display none                    # 纯无头
-display egl-headless            # GL 但无窗口（配合 VNC）
```

## 附录 B: 关键数据流时序图

```
时间 ──────────────────────────────────────────────────────────────►

vCPU线程:     GPU设备写VRAM ──────────────────────────────────────────
                                                                     │
主线程:       ···· gui_timer ····►dpy_refresh()                       │
                                      │                              │
                                      ▼                              │
                              graphic_hw_update()                     │
                                      │                              │
                                      ▼                              │
                              hw_ops->gfx_update() ◄─────────────────┘
                                      │                 (读取VRAM)
                                      ▼
                              dpy_gfx_update(x,y,w,h)
                                      │
                     ┌────────────────┼────────────────┐
                     ▼                ▼                ▼
              VNC:标记dirty     GTK:queue_draw    Spice:dirty
                     │                │                │
                     ▼                ▼                ▼
              vnc_job_push()   cairo_paint()    qxl_command()
                     │                │                │
VNC Worker:    编码(tight)           │                │
                     │                │                │
主线程:        consume_buffer()       │                │
                     │                │                │
                     ▼                ▼                ▼
              网络发送           窗口显示         SPICE传输
```

---

*文档结束*
