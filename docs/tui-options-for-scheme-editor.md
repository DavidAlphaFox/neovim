# Chez Scheme + libvim 的 TUI 选项分析

本文档分析在 Chez Scheme 中使用 libvim 作为编辑引擎时，可用的终端用户界面 (TUI) 解决方案。

## 目录

1. [背景](#背景)
2. [TUI 选项对比](#tui-选项对比)
3. [推荐架构](#推荐架构)
4. [集成模式](#集成模式)
5. [最终推荐](#最终推荐)

---

## 背景

libvim 提供了 Vim 的核心编辑功能，但**不包含 UI 渲染**。它使用回调机制让宿主应用程序处理渲染：

```
┌─────────────────────────────────────────┐
│           Host Application              │
│  ┌─────────────────────────────────┐   │
│  │    TUI Library (待选择)         │   │
│  │    - 渲染文本                    │   │
│  │    - 处理颜色                    │   │
│  │    - 响应终端事件                │   │
│  └─────────────────────────────────┘   │
│                  │                      │
│                  ▼                      │
│  ┌─────────────────────────────────┐   │
│  │         libvim                  │   │
│  │    - 缓冲区管理                  │   │
│  │    - 编辑操作                    │   │
│  │    - 模式处理                    │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

因此，选择合适的 TUI 库是构建 Scheme 原生编辑器的关键决策。

---

## TUI 选项对比

### 1. ncurses (稳定首选)

ncurses 是最成熟、最广泛使用的终端 UI 库。

#### 优点

| 特性 | 描述 |
|------|------|
| **成熟稳定** | 30+ 年历史，经过充分测试 |
| **终端兼容** | 支持几乎所有终端类型 |
| **小巧** | 库大小约 100KB |
| **Scheme 绑定** | 已有多个 Scheme ncurses 绑定 |
| **文档丰富** | 大量教程和示例 |

#### 缺点

| 特性 | 描述 |
|------|------|
| **API 繁琐** | C API 较为复杂 |
| **颜色限制** | 某些终端颜色支持有限 |
| **无内置组件** | 没有按钮、列表等高级组件 |

#### Chez Scheme FFI 示例

```scheme
;; ncurses FFI 绑定示例
(define ncurses-initscr
  (foreign-procedure "initscr" () void*))

(define ncurses-endwin
  (foreign-procedure "endwin" () int))

(define ncurses-refresh
  (foreign-procedure "refresh" () int))

(define ncurses-mvprintw
  (foreign-procedure "mvprintw" (int int string) int))

(define ncurses-getch
  (foreign-procedure "getch" () int))

(define ncurses-cbreak
  (foreign-procedure "cbreak" () int))

(define ncurses-noecho
  (foreign-procedure "noecho" () int))

(define ncurses-start-color
  (foreign-procedure "start_color" () int))

(define ncurses-init-pair
  (foreign-procedure "init_pair" (short short short) int))

;; 使用示例
(define (init-editor)
  (ncurses-initscr)
  (ncurses-cbreak)
  (ncurses-noecho)
  (ncurses-start-color)
  (ncurses-init-pair 1 2 0)  ; 绿色前景，黑色背景
  (ncurses-mvprintw 0 0 "Scheme Vim Editor")
  (ncurses-refresh))

(define (cleanup-editor)
  (ncurses-endwin))
```

---

### 2. libtickit (现代推荐)

libtickit 是由 Onivim 团队开发的现代 TUI 库，与 libvim 来自同一生态系统。

#### 优点

| 特性 | 描述 |
|------|------|
| **现代设计** | 事件驱动架构 |
| **24位颜色** | 原生支持真彩色 |
| **Unicode** | 优秀的 Unicode 处理 |
| **libvim 兼容** | 同一团队开发，API 协调 |
| **轻量** | 比其他现代库更小 |

#### 缺点

| 特性 | 描述 |
|------|------|
| **较新** | 不如 ncurses 成熟 |
| **示例少** | 文档和示例相对较少 |
| **社区小** | 使用者较少 |

#### Chez Scheme FFI 示例

```scheme
;; libtickit FFI 绑定示例
(define tickit-term-new
  (foreign-procedure "tickit_term_new" () void*))

(define tickit-term-destroy
  (foreign-procedure "tickit_term_destroy" (void*) void))

(define tickit-term-print
  (foreign-procedure "tickit_term_print" (void* string) int))

(define tickit-term-goto
  (foreign-procedure "tickit_term_goto" (void* int int) int))

(define tickit-term-chpen
  (foreign-procedure "tickit_term_chpen" (void* void*) int))

(define tickit-term-wait-input
  (foreign-procedure "tickit_term_wait_input" (void* long) int))

(define tickit-term-input-wait
  (foreign-procedure "tickit_term_input_wait" (void* int) int))

;; 使用示例
(define (init-tickit)
  (let ([term (tickit-term-new)])
    (tickit-term-print term "Scheme Vim Editor")
    term))

(define (render-line term row col text fg-color)
  (tickit-term-goto term row col)
  ;; 设置颜色
  (tickit-term-chpen term (make-pen fg-color))
  (tickit-term-print term text))
```

#### libtickit 核心概念

```scheme
;; Pen 对象 - 控制文本样式
(define tickit-pen-new
  (foreign-procedure "tickit_pen_new" () void*))

(define tickit-pen-set-colour
  (foreign-procedure "tickit_pen_set_colour_attr" (void* int int) void))

;; Window 对象 - 管理屏幕区域
(define tickit-window-new
  (foreign-procedure "tickit_window_new" (void* void*) void*))

;; 事件循环
(define (tickit-event-loop term callback)
  (let loop ()
    (let ([event (tickit-term-input-wait term -1)])
      (callback event)
      (loop))))
```

---

### 3. notcurses (功能最丰富)

notcurses 是最现代化的终端 UI 库，支持图像、视频等高级功能。

#### 优点

| 特性 | 描述 |
|------|------|
| **24位颜色** | 完整的真彩色支持 |
| **图像支持** | 可在终端显示图像和视频 |
| **高性能** | 优化的渲染引擎 |
| **丰富 API** | 大量高级功能 |
| **优秀文档** | 详细的手册和示例 |

#### 缺点

| 特性 | 描述 |
|------|------|
| **体积大** | 库大小约 2MB |
| **依赖多** | 需要更多系统依赖 |
| **过于强大** | 对于文本编辑器可能过于复杂 |
| **学习曲线** | API 较为复杂 |

#### Chez Scheme FFI 示例

```scheme
;; notcurses FFI 绑定示例
(define notcurses-init
  (foreign-procedure "notcurses_init" (void* void*) void*))

(define notcurses-stop
  (foreign-procedure "notcurses_stop" (void*) void))

(define notcurses-stdplane
  (foreign-procedure "notcurses_stdplane" (void*) void*))

(define ncplane-printf
  (foreign-procedure "ncplane_printf" (void* string) int))

(define ncplane-putegc
  (foreign-procedure "ncplane_putegc" (void* string void*) int))

(define ncplane-set-fg-rgb
  (foreign-procedure "ncplane_set_fg_rgb" (void* uint32) int))

(define ncplane-move-yx
  (foreign-procedure "ncplane_move_yx" (void* int int) int))

;; 使用示例
(define (init-notcurses)
  (let ([nc (notcurses-init #f #f)])
    (let ([stdplane (notcurses-stdplane nc)])
      (ncplane-set-fg-rgb stdplane #x00ff00)  ; 绿色
      (ncplane-printf stdplane "Scheme Vim Editor")
      nc)))

(define (render-notcurses nc)
  (let ([stdplane (notcurses-stdplane nc)])
    ;; 渲染缓冲区内容
    (ncplane-move-yx stdplane 1 0)
    (ncplane-printf stdplane "Line 1 content")))
```

---

### 4. 自定义 ANSI 转义序列 (零依赖)

直接使用 ANSI 转义序列控制终端，无需外部库。

#### 优点

| 特性 | 描述 |
|------|------|
| **零依赖** | 不需要任何外部库 |
| **完全控制** | 对每个细节的精确控制 |
| **极小体积** | 无额外开销 |
| **可移植** | 任何支持 ANSI 的终端 |

#### 缺点

| 特性 | 描述 |
|------|------|
| **手动处理** | 必须处理所有边缘情况 |
| **兼容问题** | 不同终端行为可能不同 |
| **开发量大** | 需要更多开发时间 |
| **功能有限** | 高级功能需要自己实现 |

#### Chez Scheme 实现

```scheme
;; ANSI 转义序列常量
(define ANSI-ESC #\x1b)

(define (ansi-esc . args)
  (apply string-append 
         (cons (string ANSI-ESC #\[) 
               args)))

;; 光标控制
(define (clear-screen)
  (display (ansi-esc "2J")))

(define (clear-line)
  (display (ansi-esc "2K")))

(define (move-cursor row col)
  (printf "~a~a;~aH" 
          (string ANSI-ESC #\[) row col))

(define (save-cursor)
  (display (ansi-esc "s")))

(define (restore-cursor)
  (display (ansi-esc "u")))

(define (hide-cursor)
  (display (ansi-esc "?25l")))

(define (show-cursor)
  (display (ansi-esc "?25h")))

;; 颜色控制 (256色)
(define (set-fg-color-256 n)
  (printf "~a38;5;~am" (string ANSI-ESC #\[) n))

(define (set-bg-color-256 n)
  (printf "~a48;5;~am" (string ANSI-ESC #\[) n))

;; 颜色控制 (24位真彩色)
(define (set-fg-color-rgb r g b)
  (printf "~a38;2;~a;~a;~am" 
          (string ANSI-ESC #\[) r g b))

(define (set-bg-color-rgb r g b)
  (printf "~a48;2;~a;~a;~am" 
          (string ANSI-ESC #\[) r g b))

;; 重置
(define (reset-style)
  (display (ansi-esc "0m")))

(define (bold)
  (display (ansi-esc "1m")))

(define (underline)
  (display (ansi-esc "4m")))

(define (reverse-video)
  (display (ansi-esc "7m")))

;; 完整的编辑器渲染示例
(define (render-buffer-line line row width)
  (move-cursor row 1)
  (set-fg-color-256 255)  ; 白色文本
  (set-bg-color-256 235)  ; 深灰背景
  (let ([padded (string-append line (make-string (- width (string-length line)) #\space))])
    (display (substring padded 0 (min width (string-length padded)))))
  (reset-style))

(define (render-status-line text width row)
  (move-cursor row 1)
  (set-fg-color-256 0)    ; 黑色文本
  (set-bg-color-256 75)   ; 蓝色背景
  (let ([padded (string-append text (make-string (- width (string-length text)) #\space))])
    (display (substring padded 0 (min width (string-length padded)))))
  (reset-style))

;; 输入处理
(define (read-key)
  (let ([c (read-char)])
    (cond
      [(char=? c ANSI-ESC)
       (if (char=? (peek-char) #\[)
           (begin
             (read-char)  ; 消耗 [
             (let ([next (read-char)])
               (case next
                 [(#\A) 'up]
                 [(#\B) 'down]
                 [(#\C) 'right]
                 [(#\D) 'left]
                 [else (list 'escape next)])))
           'escape)]
      [else c])))
```

---

## 推荐架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                Chez Scheme 编辑器应用                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Scheme 层                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │   │
│  │  │  插件系统    │  │  配置系统    │  │ 命令系统  │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   FFI 绑定层                         │   │
│  │  ┌─────────────────────────┐  ┌──────────────────┐  │   │
│  │  │      libvim 绑定        │  │   TUI 库绑定     │  │   │
│  │  │  - vimInput()           │  │  - 初始化        │  │   │
│  │  │  - vimBufferGetLine()   │  │  - 渲染          │  │   │
│  │  │  - vimCursorGetLine()   │  │  - 输入处理      │  │   │
│  │  │  - 回调注册             │  │  - 事件循环      │  │   │
│  │  └─────────────────────────┘  └──────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     C 库层                           │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐   │   │
│  │  │       libvim        │  │    ncurses/tickit   │   │   │
│  │  │  - 缓冲区管理       │  │  - 终端渲染         │   │   │
│  │  │  - 编辑操作         │  │  - 颜色处理         │   │   │
│  │  │  - 模式处理         │  │  - 输入捕获         │   │   │
│  │  │  - 撤销/重做        │  │  - 窗口管理         │   │   │
│  │  └─────────────────────┘  └─────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     终端                             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 数据流

```
┌─────────┐     ┌──────────┐     ┌─────────┐     ┌──────────┐
│  用户   │────▶│  终端    │────▶│  TUI库  │────▶│  事件    │
│  输入   │     │  (键盘)  │     │  (捕获) │     │  队列    │
└─────────┘     └──────────┘     └─────────┘     └──────────┘
                                                       │
                                                       ▼
┌─────────┐     ┌──────────┐     ┌─────────┐     ┌──────────┐
│  屏幕   │◀────│  TUI库   │◀────│  渲染   │◀────│  编辑器  │
│  显示   │     │  (绘制)  │     │  引擎   │     │  核心    │
└─────────┘     └──────────┘     └─────────┘     └──────────┘
                                                       │
                                                       ▼
                                                  ┌──────────┐
                                                  │  libvim  │
                                                  │  (状态)  │
                                                  └──────────┘
```

---

## 集成模式

### 模式 1: 回调驱动

libvim 通过回调通知宿主应用程序状态变化：

```scheme
;; 注册 libvim 回调
(define (setup-libvim-callbacks)
  ;; 缓冲区变化回调
  (vim-buffer-set-callback
    (lambda (buf event)
      (case event
        [(lines-changed) (render-buffer buf)]
        [(cursor-moved) (update-cursor-position buf)])))
  
  ;; 状态变化回调
  (vim-status-set-callback
    (lambda (status)
      (render-status-line status))))

;; 主循环
(define (editor-main-loop)
  (setup-libvim-callbacks)
  (let loop ()
    ;; 1. 从 TUI 获取输入
    (let ([key (tui-get-key)])
      ;; 2. 发送到 libvim
      (vim-input key)
      ;; 3. libvim 回调会触发渲染
      )
    (loop)))
```

### 模式 2: 事件循环

使用 TUI 库的事件循环作为主循环：

```scheme
;; 使用 libtickit 事件循环
(define (run-editor)
  (let* ([term (tickit-term-new)]
         [running #t])
    
    ;; 设置输入处理
    (tickit-term-set-input-callback
      term
      (lambda (event)
        (cond
          [(eq? event 'ctrl-c) 
           (set! running #f)]
          [else
           (vim-input (event-to-vim-key event))])))
    
    ;; 设置渲染回调
    (vim-set-render-callback
      (lambda ()
        (render-all-buffers term)))
    
    ;; 运行事件循环
    (let loop ()
      (when running
        (tickit-term-wait-input term 100)  ; 100ms 超时
        (loop)))
    
    ;; 清理
    (tickit-term-destroy term)))
```

### 模式 3: 协程驱动

使用 Chez Scheme 的续延 (continuation) 实现协程：

```scheme
;; 使用 call/cc 实现协程式事件处理
(define (make-coroutine proc)
  (let ([cont #f])
    (lambda ()
      (call/cc
        (lambda (k)
          (if cont
              (cont #f)
              (proc (lambda (v)
                      (call/cc 
                        (lambda (k2)
                          (set! cont k2)
                          (k v)))))))))))

;; 编辑器协程
(define editor-coroutine
  (make-coroutine
    (lambda (yield)
      (let loop ()
        (let ([key (tui-get-key)])
          (vim-input key)
          (yield 'waiting)
          (loop)))))

;; 渲染协程
(define render-coroutine
  (make-coroutine
    (lambda (yield)
      (let loop ()
        (render-all)
        (yield 'waiting)
        (loop)))))

;; 主调度器
(define (scheduler)
  (let loop ()
    (editor-coroutine)
    (render-coroutine)
    (loop)))
```

---

## 最终推荐

### 推荐方案: libtickit

基于以下理由，推荐使用 **libtickit** 作为 TUI 库：

| 评估维度 | libtickit | ncurses | notcurses | 自定义 ANSI |
|---------|-----------|---------|-----------|-------------|
| 与 libvim 兼容性 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| 现代 API 设计 | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| 24位颜色支持 | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Unicode 支持 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 库大小 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 成熟度 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Scheme 友好 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 开发效率 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |

### 推荐理由

1. **生态系统一致性**: libtickit 和 libvim 都来自 Onivim 团队，API 设计理念一致

2. **现代特性**: 
   - 原生 24 位颜色支持
   - 优秀的 Unicode 处理
   - 事件驱动架构

3. **Scheme 哲学匹配**:
   - 函数式 API 设计
   - 回调驱动模式
   - 轻量级实现

4. **合理的权衡**:
   - 比 ncurses 更现代
   - 比 notcurses 更轻量
   - 比自定义 ANSI 更可靠

### 备选方案

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 追求最大稳定性 | ncurses | 30年历史，经过充分验证 |
| 追求最大功能 | notcurses | 图像、视频等高级特性 |
| 追求最小依赖 | 自定义 ANSI | 零外部依赖 |
| **推荐平衡** | **libtickit** | **最佳功能/复杂度比** |

---

## 实现路线图

### 阶段 1: 基础绑定 (2-3 周)

```
├── libvim FFI 绑定
│   ├── 核心函数绑定
│   ├── 回调注册
│   └── 基本测试
│
└── libtickit FFI 绑定
    ├── 初始化/清理
    ├── 基本渲染
    └── 输入处理
```

### 阶段 2: 核心集成 (3-4 周)

```
├── 事件循环
│   ├── 输入 → libvim
│   ├── libvim 回调 → 渲染
│   └── 状态同步
│
├── 缓冲区渲染
│   ├── 行渲染
│   ├── 光标显示
│   └── 高亮支持
│
└── 状态栏
    ├── 模式显示
    ├── 文件名
    └── 位置信息
```

### 阶段 3: 高级功能 (4-6 周)

```
├── 多窗口
│   ├── 分屏支持
│   ├── 窗口切换
│   └── 独立缓冲区
│
├── 插件系统
│   ├── Scheme 插件 API
│   ├── 命令注册
│   └── 自动命令
│
└── 配置系统
    ├── 初始化文件
    ├── 选项设置
    └── 键映射
```

---

## libtickit vs ncurses 深度对比

### 核心对比表

| 维度 | libtickit | ncurses |
|------|-----------|---------|
| **诞生时间** | 2013 年 | 1993 年 (30+ 年) |
| **设计理念** | 现代事件驱动 | 传统过程式 |
| **架构** | 回调/事件驱动 | 直接渲染 |
| **API 风格** | 结构体 + 函数 | 全局变量 + 函数 |
| **双缓冲** | 内置自动 | 需手动实现 |
| **UTF-8 支持** | 原生 `wchar_t` | 手动编码转换 |
| **异步输入** | 非阻塞等待 | 阻塞轮询 |

---

### libtickit 优势

| 优势 | 说明 | 示例 |
|------|------|------|
| **现代 API** | 事件驱动，告别 `refresh()` 地狱 | `tickit_term_input_wait()` |
| **自动双缓冲** | 不会看到屏幕闪烁 | 内置实现 |
| **UTF-8 原生** | 直接用 `wchar_t` | 无需手动转换 |
| **异步输入** | 可以非阻塞等待按键 | `tickit_term_input_wait_nonblocking()` |
| **窗口堆叠** | 支持父子窗口关系 | `tickit_window_new()` |
| **Pen 系统** | 统一的样式对象 | `tickit_pen_new()` |
| **子进程管道** | 更易集成外部命令 | 内置支持 |

#### 代码对比：事件循环

```c
// ncurses: 传统的阻塞/轮询
while (running) {
    int ch = getch();  // 阻塞直到按键
    handle_input(ch);
    refresh();  // 手动刷新
}

// libtickit: 回调驱动的非阻塞
tickit_term_set_input_callback(term, handler, user_data);
tickit_run(term);  // 内部事件循环，自动调用 handler
```

#### 代码对比：Hello World

```c
// ncurses
initscr();
mvprintw(0, 0, "Hello");
refresh();
getch();
endwin();

// libtickit
TickitTerm *term = tickit_term_new();
tickit_term_print(term, "Hello");
tickit_term_input_wait(term, -1);  // 等待任意输入
tickit_term_destroy(term);
```

---

### libtickit 劣势

| 劣势 | 说明 | 影响 |
|------|------|------|
| **不兼容 curses** | 不是 curses 替代品，是全新设计 | 无法直接用 ncurses 代码 |
| **生态小** | 几乎只有 Onivim 在用 | 难以找到示例和帮助 |
| **文档少** | 没有 ncurses 那么多书籍/教程 | 学习曲线陡峭 |
| **需自己管理布局** | 没有 `WINDOW*` 坐标系统 | 开发工作量大 |
| **调试困难** | 回调嵌套，调用栈复杂 | 调试体验差 |
| **社区支持少** | 遇到问题难以找到答案 | 风险高 |

---

### ncurses 优势

| 优势 | 说明 | 影响 |
|------|------|------|
| **30+ 年成熟** | 经过充分测试和生产验证 | 稳定可靠 |
| **广泛兼容** | 支持几乎所有终端类型 | 最低终端兼容风险 |
| **丰富生态** | 大量教程、书籍、示例代码 | 容易学习和开发 |
| **成熟绑定** | 多种语言都有成熟绑定 | Scheme 已有实现 |
| **简单直接** | 过程式 API，易理解 | 快速开发 |
| **调试友好** | 函数调用链清晰 | 容易定位问题 |

---

### 实际项目使用情况

| 项目 | TUI 库 | 说明 |
|------|--------|------|
| **Onivim** | libtickit | libvim 的 GUI，官方配套 |
| **libvim 测试** | libtickit | libvim 自带的测试用例 |
| **Neovim TUI** | ncurses | 复用 Vim 的 TUI 代码 |
| **大多数终端工具** | ncurses | GNU coreutils, htop, vim 等 |

---

### 关键差异：事件循环模型

```
ncurses 方式:
┌────────────────────────────────────────┐
│  main loop:                            │
│    ┌─────────────────────────────┐     │
│    │ ch = getch()  // 阻塞       │     │
│    └─────────────────────────────┘     │
│                │                       │
│                ▼                       │
│    ┌─────────────────────────────┐     │
│    │ handle_input(ch)           │     │
│    └─────────────────────────────┘     │
│                │                       │
│                ▼                       │
│    ┌─────────────────────────────┐     │
│    │ refresh()  // 手动刷新     │     │
│    └─────────────────────────────┘     │
└────────────────────────────────────────┘

libtickit 方式:
┌────────────────────────────────────────┐
│  tickit_set_input_callback(            │
│    term, callback, data               │
│  );                                   │
│  tickit_run(term);  // 内部事件循环   │
│                                        │
│  callback:                            │
│    ┌─────────────────────────────┐     │
│    │ void handler(int event) {   │     │
│    │   vim_input(event);         │     │
│    │   // libvim 触发渲染回调    │     │
│    │ }                           │     │
│    └─────────────────────────────┘     │
└────────────────────────────────────────┘
```

---

### 在 Chez Scheme 中的集成复杂度

```scheme
;; ncurses Chez Scheme FFI (简单直接)
(define ncurses-initscr
  (foreign-procedure "initscr" () void*))

(define ncurses-mvprintw
  (foreign-procedure "mvprintw" (int int string) int))

(define ncurses-getch
  (foreign-procedure "getch" () int))

;; 使用: 简单函数调用
(ncurses-initscr)
(ncurses-mvprintw 0 0 "Hello")
(let ([ch (ncurses-getch)])
  (handle-key ch))

;; libtickit Chez Scheme FFI (需要处理回调)
(define tickit-term-new
  (foreign-procedure "tickit_term_new" () void*))

(define tickit-term-set-input-callback
  (foreign-procedure "tickit_term_set_input_callback" 
    (void* void* void*) void))  ;; 需要传递回调函数指针

;; 使用: 需要将 Scheme 过程注册为 C 回调
;; 这需要复杂的 C 桥接代码
```

---

## Chez Scheme FFI 绑定挑战深度分析

### 核心挑战对比

| 挑战 | libtickit | ncurses |
|------|-----------|---------|
| **回调处理** | ❌ 核心机制，需 C 桥接 | ✅ 可选，较少 |
| **事件循环** | ❌ `tickit_run()` 需适配 | ✅ 自己控制循环 |
| **FFI 复杂度** | 高 | 中 |
| **状态管理** | 结构体传递 | 全局变量 |

---

### libtickit + Chez Scheme FFI 挑战

#### 1. 回调问题 (最大障碍)

libtickit 核心是回调机制，但 Scheme 无法直接将过程传递给 C：

```c
// libtickit C API: 传递函数指针
void tickit_term_set_input_callback(TickitTerm *t, 
    int (*callback)(TickitEvent event, void *info, void *data), 
    void *data);

// 问题: 如何将 Scheme 过程转换为 C 函数指针？
```

**解决方案**: 需要 C trampoline

```c
// C 桥接代码 (必须用 C，不能纯 Scheme)
static int scheme_callback_wrapper(TickitEvent event, void *info, void *data) {
    Scheme_Object *proc = (Scheme_Object *)data;
    // 调用 scheme_apply(proc, ...)
    return result;
}

// 然后注册
tickit_term_set_input_callback(term, scheme_callback_wrapper, scheme_proc);
```

**问题**:
- 必须编写 C 桥接代码
- 需要管理回调的生命周期
- 调试困难 (C ↔ Scheme 调用栈混杂)

#### 2. 事件循环冲突

```c
// libtickit 有自己的事件循环
tickit_run(term);  // 阻塞直到退出

// 但 Scheme 也可能有自己的调度器
// 两者如何共存？
```

**可选方案**:

| 方案 | 描述 | 复杂度 |
|------|------|--------|
| **放弃 tickit_run** | 自己用 select/poll 轮询 | 高 |
| **tickit_run 作为主循环** | Scheme 只做计算 | 中 |
| **协程桥接** | 用 call/cc 模拟 | 非常高 |

#### 3. 复杂类型管理

```c
// libtickit 多个对象类型
Tickit *t;
TickitTerm *term;
TickitWindow *win;
TickitPen *pen;

// 每个都需要在 Scheme 中表示为指针
// 并确保正确释放
```

#### libtickit FFI 绑定完整示例

```scheme
;; 需要 C 桥接代码 + Scheme FFI

;; 1. C 文件: tickit_callbacks.c
;; 必须预先注册回调类型

;; 2. Scheme FFI
(define tickit-term-new
  (foreign-procedure "tickit_term_new" () void*))

(define tickit-term-set-input-callback!
  (foreign-procedure "tickit_term_set_input_callback" 
    (void* void* void*) void))
;; 第二个 void* 是 C 函数指针，无法直接从 Scheme 传递！

;; 3. 必须用 C 创建 trampoline
;; 然后在 Scheme 端调用 C 函数注册
(define (register-input-handler! term scheme-proc)
  (let ([c-trampoline (get-callback-trampoline scheme-proc)])
    (tickit-term-set-input-callback! term 
      (address-of-c-trampoline c-trampoline)
      c-trampoline)))
```

---

### ncurses + Chez Scheme FFI 优势

#### 1. 简单直接的函数

```scheme
;; ncurses 函数签名简单
(define initscr (foreign-procedure "initscr" () void*))
(define endwin (foreign-procedure "endwin" () int))
(define refresh (foreign-procedure "refresh" () int))
(define getch (foreign-procedure "getch" () int))
(define mvprintw (foreign-procedure "mvprintw" (int int string) int))

;; 直接调用，无需回调
(initscr)
(mvprintw 0 0 "Hello, Scheme!")
(refresh)
```

#### 2. 状态通过参数传递

```scheme
;; ncurses 的 WINDOW* 可以作为参数
(define mvwaddch 
  (foreign-procedure "mvwaddch" (void* int int char) int))

;; 可以跟踪窗口对象
(let ([win (newwin 10 10 0 0)])
  (mvwaddch win 5 5 #\x))
```

#### 3. 已有参考实现

```
Scheme ncurses 绑定:
- Racket: raco pkg install ncurses
- Gambit: 有现成例子
- CHICKEN: chicken-ncurses
```

---

### ncurses 绑定挑战

#### 1. 阻塞的 getch

```scheme
;; getch() 会阻塞主线程
(let ([ch (getch)])  ;; 阻塞！
  (case ch
    [(#\q) (quit)]
    [else (handle ch)]))
```

**解决方案**: 用 Scheme 协程

```scheme
;; 用 call/cc 实现非阻塞
(define (get-key-nonblocking)
  (call/cc
    (lambda (k)
      ;; 设置中断返回点
      (set! key-cont k)
      (when (kbhit)
        (k (getch))))))

;; 在事件循环中调用
(define (event-loop)
  (when (kbhit)
    (let ([key (getch)])
      (handle-key key)))
  (sleep 0.01)  ;; 让出控制权
  (event-loop))
```

#### 2. 全局状态

```scheme
;; ncurses 内部有全局状态
;; 需要确保正确清理
(define (with-ncurses thunk)
  (dynamic-wind
    (lambda () (initscr) (cbreak) (noecho))
    thunk
    (lambda () (endwin))))
```

---

### FFI 复杂度对比

#### libtickit (高复杂度)

```
┌─────────────────────────────────────────────────────────┐
│                    绑定层级                              │
├─────────────────────────────────────────────────────────┤
│  Scheme 代码                                            │
│      │                                                  │
│      ▼                                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  C trampoline 函数 (必须用 C 编写)              │   │
│  │  - 输入回调桥接                                  │   │
│  │  - 渲染回调桥接                                  │   │
│  │  - 生命周期管理                                  │   │
│  └─────────────────────────────────────────────────┘   │
│      │                                                  │
│      ▼                                                  │
│  libtickit C 库                                        │
│      │                                                  │
│      ▼                                                  │
│  终端                                                  │
└─────────────────────────────────────────────────────────┘

开发工作:
├── 编写 C 桥接代码 (tickit_callbacks.c)
├── 编写 Scheme FFI 绑定
├── 处理回调生命周期
├── 处理事件循环冲突
└── 调试 C↔Scheme 混合调用栈

预估时间: 4-8 周
```

#### ncurses (中复杂度)

```
┌─────────────────────────────────────────────────────────┐
│                    绑定层级                              │
├─────────────────────────────────────────────────────────┤
│  Scheme 代码                                            │
│      │                                                  │
│      ▼                                                  │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Scheme FFI 绑定 (纯 Scheme)                     │   │
│  │  - 函数声明                                      │   │
│  │  - 类型转换                                      │   │
│  │  - 错误处理                                      │   │
│  └─────────────────────────────────────────────────┘   │
│      │                                                  │
│      ▼                                                  │
│  ncurses C 库                                          │
│      │                                                  │
│      ▼                                                  │
│  终端                                                  │
└─────────────────────────────────────────────────────────┘

开发工作:
├── 编写 Scheme FFI 绑定 (纯 Scheme)
├── 处理全局状态
├── 实现非阻塞输入 (用协程)
└── 调试纯 Scheme 调用栈

预估时间: 1-2 周
```

---

### 复杂度总结

| 维度 | libtickit | ncurses |
|------|-----------|---------|
| **FFI 复杂度** | 高 (需要 C 桥接) | 中 (纯 Scheme) |
| **回调处理** | 必须用 C trampoline | 可选 |
| **事件循环** | 需适配 tickit_run | 自己控制 |
| **开发时间** | 4-8 周 | 1-2 周 |
| **维护成本** | 高 | 低 |
| **调试难度** | 高 | 低 |
| **参考实现** | 几乎没有 | Racket, Gambit, CHICKEN 已有 |

---

###Chez Scheme FFI 绑定最终建议

#### 如果追求快速开发: **ncurses**

- FFI 简单，纯 Scheme 可完成绑定
- 已有多种 Scheme 的参考实现
- 调试友好，维护成本低
- 1-2 周可完成基础绑定

#### 如果与 libvim 配套: 需要权衡

| 方案 | 描述 | 风险 |
|------|------|------|
| **方案 A** | 直接用 libtickit | 高: 4-8 周 FFI 开发 |
| **方案 B** | ncurses 原型 + 评估 | 中: 先验证可行性 |
| **方案 C** | 用 libvim 自带的测试代码 | 低: 但功能有限 |

**推荐方案 B**: 
1. 先用 ncurses 实现原型 (1-2 周)
2. 验证 libvim 集成是否可行
3. 如果可行，再评估是否值得迁移到 libtickit

---

### 总结建议

| 场景 | 推荐 | 理由 |
|------|------|------|
| 与 libvim 配套 | **libtickit** | 同一团队，API 协调 |
| 需要最大兼容性 | **ncurses** | 30 年验证，支持所有终端 |
| 需要稳定成熟 | **ncurses** | 生产验证，故障少 |
| 需要现代异步 API | libtickit | 事件驱动架构 |
| 需要避免闪烁 | libtickit | 自动双缓冲 |
| 快速开发 | ncurses | 简单直接，生态丰富 |

#### 最终建议

**如果与 libvim 配套**: libtickit 是更好的选择
- 同一团队开发，API 设计协调
- Onivim 已有成功案例
- 事件驱动与 libvim 的回调机制匹配

**如果需要最大稳定性和兼容性**: ncurses 更安全
- 经过 30 年生产验证
- 遇到问题容易找到解决方案
- Scheme 绑定更成熟

---

## 参考资源

### libtickit

- GitHub: https://github.com/leonerd/libtickit
- 文档: https://www.leonerd.org.uk/code/libtickit/

### ncurses

- 官网: https://invisible-island.net/ncurses/
- 手册: https://invisible-island.net/ncurses/ncurses.html

### notcurses

- GitHub: https://github.com/dankamongmen/notcurses
- 文档: https://notcurses.com/

### ANSI 转义序列

- 维基百科: https://en.wikipedia.org/wiki/ANSI_escape_code
- 参考: https://gist.github.com/fnky/458719343aabd01cfb17a3a4f7296797
