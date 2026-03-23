# 反向架构：将 Vim 嵌入 Chez Scheme

> 本文档分析将 Vim/Neovim 的编辑功能作为库嵌入 Chez Scheme 的可行性，这是"将 Chez Scheme 嵌入 Neovim"的反向思路。

## 目录

1. [核心发现](#1-核心发现)
2. [已有方案对比](#2-已有方案对比)
3. [架构设计](#3-架构设计)
4. [技术实现](#4-技术实现)
5. [与原方案对比](#5-与原方案对比)
6. [Scheme 编辑器先例](#6-scheme-编辑器先例)
7. [最小功能集](#7-最小功能集)
8. [可行性评估](#8-可行性评估)
9. [实现路线图](#9-实现路线图)

---

## 1. 核心发现

```
┌─────────────────────────────────────────────────────────────────┐
│  反向架构可行性: ★★★★☆ (4/5) — 比原方案更可行                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  关键发现:                                                       │
│  1. ✅ libvim 已存在 — 专为嵌入设计的 Vim 引擎                   │
│  2. ✅ 本 fork 已有 libnvim — 被 VimR (Swift GUI) 使用           │
│  3. ✅ Chez Scheme FFI 成熟 — 可直接调用 C 函数                  │
│  4. ✅ 有先例可循 — Edwin, Guile-Emacs                           │
│                                                                  │
│  结论: 创建一个 Scheme-native 编辑器，使用 Vim 编辑引擎          │
│        比将 Scheme 嵌入 Neovim 更可行                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 已有方案对比

### 2.1 Vim 作为库的项目

| 项目 | 语言 | 状态 | 用途 |
|------|------|------|------|
| **libvim** | C | ✅ 活跃 | Onivim 的 Vim 引擎，UI 无关 |
| **libnvim** (本 fork) | C | ✅ 存在 | VimR 的静态库，msgpack-rpc |
| **vim.wasm** | WASM | ⚠️ 实验 | 浏览器中的 Vim |

### 2.2 libvim vs libnvim

| 特性 | libvim | libnvim (本 fork) |
|------|--------|-------------------|
| **来源** | Onivim 项目 | Neovim 修改版 |
| **接口** | C API (~80 函数) | msgpack-rpc |
| **UI 依赖** | 无 | 需要 UI 事件循环 |
| **维护** | 活跃 | 随 Neovim 版本 |
| **文档** | 完整 | 有限 |
| **大小** | ~500KB | ~2MB |

### 2.3 本 fork 的 libnvim 架构

```
┌─────────────────────────────────────────────────────────────────┐
│  本 fork 已有架构 (用于 VimR)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐     ┌─────────────────┐     ┌───────────┐  │
│  │   libnvim.a     │     │  NvimServer     │     │  VimR.app │  │
│  │  (静态库 ~2MB)  │────►│  (Swift RPC)    │────►│ (macOS)   │  │
│  └─────────────────┘     └─────────────────┘     └───────────┘  │
│         │                        │                              │
│         │   msgpack-rpc          │   Swift FFI                  │
│         │   over stdio           │                              │
│         ▼                        ▼                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Package.swift                         │    │
│  │  - 静态链接: libnvim.a, libluv, libuv, libvterm         │    │
│  │  - 框架: CoreFoundation                                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  关键文件:                                                       │
│  ├── NvimServer/bin/build_libnvim.sh                            │
│  ├── Package.swift (Swift 包定义)                               │
│  └── RxNeovim/Sources/RxNeovimApi.swift (API 客户端)            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 架构设计

### 3.1 Chez Scheme + libvim 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scheme-Vim 编辑器架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Chez Scheme (宿主)                     │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────┐ │   │
│  │  │   UI Layer      │  │  Plugin System  │  │  Keymaps  │ │   │
│  │  │  (Tk/GLFW/SDL)  │  │  (Pure Scheme)  │  │  (Scheme) │ │   │
│  │  └────────┬────────┘  └────────┬────────┘  └─────┬─────┘ │   │
│  │           │                    │                  │        │   │
│  │           └────────────────────┼──────────────────┘        │   │
│  │                                │                           │   │
│  │                                ▼                           │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │                 FFI 绑定层 (Scheme)                  │  │   │
│  │  │  (define vim-input (foreign-procedure ...))         │  │   │
│  │  │  (define vim-buffer-get-line ...)                   │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    libvim.a (静态库)                      │   │
│  │  ├── vimInit() / vimInput() / vimKey()                   │   │
│  │  ├── vimBufferOpen() / vimBufferGetLine()                │   │
│  │  ├── vimCursorGetPosition() / vimCursorSetPosition()     │   │
│  │  ├── vimExecute() / vimEval()                            │   │
│  │  └── 回调: buffer updates, cursor moves, mode changes    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  特点:                                                          │
│  ├── Scheme 完全控制运行时                                      │
│  ├── Vim 编辑功能作为"黑盒"引擎                                │
│  ├── UI 层完全由 Scheme 实现                                    │
│  └── 所有插件用 Scheme 编写                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 替代架构：Chez Scheme + libnvim (本 fork)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scheme-Nvim 编辑器架构                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Chez Scheme (宿主)                     │   │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐    │   │
│  │  │   UI Layer      │  │  msgpack-rpc 客户端 (Scheme) │    │   │
│  │  └────────┬────────┘  └──────────────┬──────────────┘    │   │
│  │           │                          │                    │   │
│  │           │                          ▼                    │   │
│  │           │         ┌─────────────────────────────────┐  │   │
│  │           │         │  libnvim.a + 事件循环            │  │   │
│  │           │         │  (嵌入式 Neovim 运行时)          │  │   │
│  │           │         └─────────────────────────────────┘  │   │
│  └───────────┼───────────────────────────────────────────────┘   │
│              │                                                    │
│              ▼                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    完整 Neovim 功能                       │   │
│  │  ├── LSP 客户端 (Lua)                                     │   │
│  │  ├── Tree-sitter (Lua)                                    │   │
│  │  ├── 诊断系统 (Lua)                                       │   │
│  │  └── 所有 Vim 编辑功能                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  优点: 保留 Neovim 的所有现代功能 (LSP, Tree-sitter)            │
│  缺点: 需要管理 libuv 事件循环，更复杂                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 技术实现

### 4.1 Chez Scheme FFI 绑定

```scheme
;; vim-ffi.ss - Chez Scheme FFI 绑定到 libvim

;; 加载共享库
(load-shared-object "./libvim.so")

;; 基础类型定义
(define-ftype vim-buffer-t void*)
(define-ftype vim-cursor-t void*)

;; 初始化
(define vim-init
  (foreign-procedure "vimInit" (int ptr) void))

;; 输入处理
(define vim-input
  (foreign-procedure "vimInput" (string) void))

(define vim-key
  (foreign-procedure "vimKey" (string) void))

;; Buffer 操作
(define vim-buffer-open
  (foreign-procedure "vimBufferOpen" (string) vim-buffer-t))

(define vim-buffer-get-line
  (foreign-procedure "vimBufferGetLine" (vim-buffer-t int) string))

(define vim-buffer-set-lines
  (foreign-procedure "vimBufferSetLines" 
    (vim-buffer-t int int int ptr) void))

(define vim-buffer-get-line-count
  (foreign-procedure "vimBufferGetLineCount" (vim-buffer-t) int))

;; 光标操作
(define vim-cursor-get-position
  (foreign-procedure "vimCursorGetPosition" () int))

(define vim-cursor-set-position
  (foreign-procedure "vimCursorSetPosition" (int int) void))

;; 执行命令
(define vim-execute
  (foreign-procedure "vimExecute" (string) void))

(define vim-eval
  (foreign-procedure "vimEval" (string) string))

;; 模式查询
(define vim-get-mode
  (foreign-procedure "vimGetMode" () int))
```

### 4.2 高级 Scheme API

```scheme
;; vim-api.ss - 高级 Scheme API

;; 初始化编辑器
(define (vim-start)
  (vim-init 0 #f)
  (setup-defaults))

;; 编辑操作
(define (vim-insert text)
  (vim-input "i")
  (vim-input text)
  (vim-input "\x1b"))  ; ESC

(define (vim-normal-command cmd)
  (vim-input "\x1b")  ; 确保在 Normal 模式
  (vim-input cmd))

;; Buffer 操作
(define (vim-get-current-buffer-contents)
  (let* ((buf (vim-buffer-get-current))
         (count (vim-buffer-get-line-count buf)))
    (let loop ((i 1) (lines '()))
      (if (> i count)
          (string-join (reverse lines) "\n")
          (loop (+ i 1)
                (cons (vim-buffer-get-line buf i) lines))))))

;; 搜索
(define (vim-search pattern)
  (vim-execute (format "/~a" pattern)))

;; 保存/加载
(define (vim-save-file path)
  (vim-execute (format "w ~a" path)))

(define (vim-load-file path)
  (vim-buffer-open path))

;; Ex 命令
(define (vim-command cmd)
  (vim-execute cmd))

;; 宏定义 - Vim 风格的命令组合
(define-syntax vim-do
  (syntax-rules ()
    [(_ cmd) (vim-input cmd)]
    [(_ cmd1 cmd2 ...)
     (begin (vim-input cmd1)
            (vim-do cmd2 ...))]))

;; 使用示例
(vim-start)
(vim-load-file "test.scm")
(vim-insert "(define hello \"world\")")
(vim-save-file "test.scm")
```

### 4.3 UI 层示例 (使用 Tk)

```scheme
;; vim-ui.ss - 使用 Chez Scheme Tk 绑定的 UI 层

;; 创建主窗口
(define (create-editor-window)
  (tk-init)
  (let ((frame (tk-create-frame '()))
        (text-widget (tk-create-text frame '((height . 40) (width . 80)))))
    
    ;; 绑定键盘事件
    (tk-bind text-widget '<Key>
      (lambda (event)
        (handle-key-event event)))
    
    ;; 设置 Vim 回调
    (vim-set-buffer-update-callback
      (lambda (buf line content)
        (update-text-widget text-widget content)))
    
    (tk-pack frame)
    (tk-pack text-widget)
    (tk-main-loop)))

;; 键盘事件处理
(define (handle-key-event event)
  (let ((key (event-key event)))
    (vim-input key)
    (update-display)))

;; 显示更新
(define (update-display)
  (let ((content (vim-get-current-buffer-contents))
        (cursor-pos (vim-cursor-get-position)))
    (update-text-widget content)
    (update-cursor cursor-pos)))
```

### 4.4 回调处理

```scheme
;; vim-callbacks.ss - 处理 Vim 到 Scheme 的回调

;; Buffer 更新回调
(define-ftype buffer-update-cb
  (function (vim-buffer-t int int string) void))

(define (register-buffer-callback)
  (let ((cb (foreign-callable 
              (lambda (buf start end content)
                (scheme-handle-buffer-update buf start end content))
              (vim-buffer-t int int string)
              void)))
    (lock-object cb)
    (vim-register-buffer-callback 
      (foreign-callable-entry-point cb))))

;; 模式变化回调
(define (register-mode-callback)
  (let ((cb (foreign-callable
              (lambda (new-mode)
                (scheme-handle-mode-change new-mode))
              (int)
              void)))
    (lock-object cb)
    (vim-register-mode-callback
      (foreign-callable-entry-point cb))))

;; Scheme 端处理
(define (scheme-handle-buffer-update buf start end content)
  (update-ui-buffer buf content)
  (run-buffer-hooks buf start end content))

(define (scheme-handle-mode-change new-mode)
  (update-mode-line new-mode)
  (run-mode-hooks new-mode))
```

---

## 5. 与原方案对比

### 5.1 架构对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    两种方案对比                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  原方案: 将 Chez Scheme 嵌入 Neovim                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Neovim (宿主)                        │    │
│  │  ├── C 核心                                              │    │
│  │  ├── Lua 运行时                                          │    │
│  │  ├── Vimscript 运行时                                    │    │
│  │  └── Chez Scheme (嵌入式)  ◀── 新增                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  问题:                                                          │
│  ├── 事件循环集成复杂                                          │
│  ├── GC 交互困难                                               │
│  ├── 生态归零 (所有 Lua 插件丢失)                              │
│  └── 维护负担重                                                │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  反向方案: 将 Vim 嵌入 Chez Scheme                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   Chez Scheme (宿主)                     │    │
│  │  ├── Scheme 运行时                                       │    │
│  │  ├── UI 层 (Tk/GLFW/...)                                 │    │
│  │  ├── 插件系统 (Scheme)                                   │    │
│  │  └── libvim (嵌入式 Vim 引擎)  ◀── 编辑核心              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  优点:                                                          │
│  ├── Scheme 完全控制运行时                                      │
│  ├── FFI 集成简单 (直接调用 C)                                 │
│  ├── 创建新产品 (不是修改 Neovim)                              │
│  └── 渐进式开发                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 详细对比表

| 方面 | Scheme 嵌入 Neovim | Vim 嵌入 Scheme |
|------|-------------------|-----------------|
| **控制权** | Neovim 主导 | Scheme 主导 |
| **启动** | 需要 Neovim | 独立 Scheme 程序 |
| **内存** | 两个 GC 协调 | Scheme GC 独立 |
| **事件循环** | 需要集成到 libuv | Scheme 自由选择 |
| **UI** | Neovim TUI/GUI | 自定义 (Tk/GLFW) |
| **插件语言** | Lua + Scheme | 纯 Scheme |
| **生态** | 保留 Neovim 插件 | 全新生态 |
| **复杂度** | ★★★★★ (高) | ★★★☆☆ (中) |
| **维护** | 跟踪 Neovim 变化 | 跟踪 libvim 变化 |

### 5.3 功能保留对比

| 功能 | Scheme 嵌入 Neovim | Vim 嵌入 Scheme |
|------|-------------------|-----------------|
| **Vim 编辑模式** | ✅ 完整 | ✅ 完整 (via libvim) |
| **LSP** | ✅ 保留 | ❌ 需用 Scheme 重写 |
| **Tree-sitter** | ✅ 保留 | ❌ 需用 Scheme 重写 |
| **诊断系统** | ✅ 保留 | ❌ 需用 Scheme 重写 |
| **Lua 插件** | ✅ 保留 | ❌ 丢失 |
| **Vimscript** | ✅ 保留 | ⚠️ 部分 (via libvim) |
| **Scheme 插件** | ✅ 新增 | ✅ 全部 |

---

## 6. Scheme 编辑器先例

### 6.1 Edwin (MIT Scheme)

Edwin 是 MIT Scheme 自带的 Emacs 风格编辑器：

> "Edwin 与 GNU Emacs 非常相似，也有扩展语言 Emacs Lisp... 在 Edwin 中，Scheme 是与 Edwin 实现相同的方言。"

**设计教训**：
- 命令 与过程 是不同对象
- 避免用户代码与编辑器内部冲突
- 完全用 Scheme 实现是可行的

### 6.2 Guile-Emacs

将 GNU Emacs 的 Elisp 编译为 Guile Scheme：

- 保留 Emacs 兼容性
- 使用 Scheme 作为底层
- 证明了 Lisp 系编辑器的可行性

### 6.3 经验总结

```
┌─────────────────────────────────────────────────────────────────┐
│  Scheme 编辑器设计经验                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 分离关注点                                                   │
│     - 编辑器命令 vs Scheme 过程                                 │
│     - UI 层 vs 编辑逻辑                                         │
│                                                                  │
│  2. 回调机制                                                     │
│     - 使用 Scheme 的 first-class continuation                   │
│     - 事件驱动架构                                              │
│                                                                  │
│  3. 模块化                                                       │
│     - 核心编辑功能 (libvim)                                     │
│     - UI 层 (可替换)                                            │
│     - 插件系统 (Scheme 模块)                                    │
│                                                                  │
│  4. 性能                                                         │
│     - 关键路径用 C (libvim)                                     │
│     - 扩展用 Scheme                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 最小功能集

### 7.1 libvim 核心 API (~80 函数)

```c
// 初始化
void vimInit(int argc, char **argv);

// 输入
void vimInput(const char *input);
void vimKey(const char *key);

// Buffer
vimBuffer *vimBufferOpen(const char *filename, int flags);
char *vimBufferGetLine(vimBuffer *buf, int lineno);
void vimBufferSetLines(vimBuffer *buf, int start, int end, ...);
int vimBufferGetLineCount(vimBuffer *buf);
vimBuffer *vimBufferGetCurrent(void);

// 光标
void vimCursorGetPosition(int *row, int *col);
void vimCursorSetPosition(int row, int col);

// 模式
int vimGetMode(void);
int vimGetSubMode(void);

// 执行
void vimExecute(const char *cmd);
char *vimEval(const char *expr);

// 搜索
int vimSearchGetMatchingPair(int *row, int *col);

// 回调注册
void vimSetBufferUpdateCallback(callback_t cb);
void vimSetCursorMoveCallback(callback_t cb);
void vimSetModeChangeCallback(callback_t cb);
```

### 7.2 Scheme 包装最小集

```scheme
;; === 初始化 ===
(vim-init)                    ; 启动 Vim 引擎

;; === 输入 ===
(vim-input "i")               ; 进入 Insert 模式
(vim-input "hello")           ; 输入文本
(vim-input "\x1b")            ; ESC，返回 Normal 模式

;; === Buffer ===
(vim-buffer-open "file.scm")  ; 打开文件
(vim-buffer-get-line buf n)   ; 获取第 n 行
(vim-buffer-lines buf)        ; 获取所有行

;; === 光标 ===
(vim-cursor-pos)              ; 获取光标位置
(vim-cursor-move! row col)    ; 移动光标

;; === 命令 ===
(vim-cmd "w")                 ; 保存
(vim-cmd "q")                 ; 退出
(vim-cmd "%s/old/new/g")      ; 替换

;; === 模式 ===
(vim-mode)                    ; 获取当前模式

;; === 搜索 ===
(vim-search "pattern")        ; 搜索
```

### 7.3 完整编辑器需要的额外功能

| 类别 | 功能 | 优先级 |
|------|------|--------|
| **UI** | 窗口渲染、滚动、光标显示 | 高 |
| **高亮** | 语法高亮 (可用 Tree-sitter) | 中 |
| **补全** | 自动补全框架 | 中 |
| **文件** | 文件浏览器 | 低 |
| **Git** | 版本控制集成 | 低 |
| **LSP** | 语言服务器 (需重写) | 高 |

---

## 8. 可行性评估

### 8.1 技术可行性矩阵

| 挑战 | 可行性 | 复杂度 | 说明 |
|------|--------|--------|------|
| **FFI 绑定** | ✅ | 低 | Chez FFI 成熟 |
| **libvim 集成** | ✅ | 低 | 清晰的 C API |
| **UI 层** | ✅ | 中 | 可用 Tk/GLFW |
| **事件处理** | ✅ | 中 | Scheme 回调机制 |
| **性能** | ✅ | - | 原生调用，无开销 |
| **维护** | ⚠️ | 中 | 跟踪 libvim 变化 |

### 8.2 开发工作量估算

| 阶段 | 工作内容 | 工作量 |
|------|----------|--------|
| **Phase 1** | FFI 绑定 (~80 函数) | 2-4 周 |
| **Phase 2** | 高级 API 包装 | 1-2 周 |
| **Phase 3** | 基础 UI (Tk) | 2-4 周 |
| **Phase 4** | 插件系统 | 2-4 周 |
| **Phase 5** | 语法高亮 | 2-4 周 |
| **Phase 6** | LSP 集成 | 4-8 周 |
| **总计** | MVP | **3-6 个月** |

### 8.3 最终评估

```
┌─────────────────────────────────────────────────────────────────┐
│                    反向架构可行性评估                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  技术可行性:  ★★★★☆ (4/5)                                       │
│  ├── ✅ libvim 提供干净的 C API                                 │
│  ├── ✅ Chez Scheme FFI 成熟                                    │
│  ├── ✅ 有 Scheme 编辑器先例 (Edwin)                            │
│  └── ✅ 本 fork 已有 libnvim 参考                               │
│                                                                  │
│  实际可行性:  ★★★★☆ (4/5)                                       │
│  ├── ✅ 创建新产品，不破坏现有生态                              │
│  ├── ✅ 渐进式开发，MVP 可快速实现                              │
│  ├── ✅ Scheme 社区可能感兴趣                                   │
│  └── ⚠️ UI 层需要从零开发                                       │
│                                                                  │
│  对比原方案:                                                     │
│  ├── 复杂度降低 40%                                             │
│  ├── 风险降低 60%                                               │
│  └── 创造新价值 (Scheme-native 编辑器)                          │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  结论: 反向架构比原方案更可行                                    │
│        推荐创建一个 Scheme-native 编辑器                         │
│        使用 libvim 作为编辑引擎                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. 实现路线图

### 9.1 Phase 1: 基础绑定 (2-4 周)

```
目标: 验证 libvim + Chez Scheme FFI

任务:
├── 编译 libvim 为共享库
├── 编写 Chez FFI 绑定 (核心 20-30 函数)
├── 测试基本编辑操作
│   ├── vim-init / vim-input
│   ├── buffer 操作
│   └── cursor 操作
└── 验证回调机制

交付: vim-ffi.ss 基础模块
```

### 9.2 Phase 2: 高级 API (1-2 周)

```
目标: Scheme 友好的 API

任务:
├── 设计 Scheme 风格的 API
├── 实现宏系统 (vim-do ...)
├── 添加错误处理
└── 编写测试

交付: vim-api.ss 高级模块
```

### 9.3 Phase 3: UI 原型 (2-4 周)

```
目标: 可用的文本编辑器

任务:
├── 选择 UI 框架 (Tk/GLFW/SDL)
├── 实现文本渲染
├── 键盘事件处理
├── 光标显示
└── 基础滚动

交付: vim-ui.ss UI 模块
```

### 9.4 Phase 4: 插件系统 (2-4 周)

```
目标: 可扩展的编辑器

任务:
├── 设计插件 API
├── Hook 系统
├── 自动加载机制
└── 示例插件

交付: vim-plugin.ss 插件框架
```

### 9.5 Phase 5: 高级功能 (4-8 周)

```
目标: 现代编辑器功能

任务:
├── 语法高亮 (Tree-sitter 集成?)
├── 文件浏览器
├── 搜索/替换 UI
├── Git 集成
└── LSP 客户端 (Scheme 实现)

交付: 功能完整的编辑器
```

### 9.6 长期愿景

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scheme-Vim 愿景                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  短期 (6 个月):                                                  │
│  ├── 可用的 Scheme 编辑器                                       │
│  ├── Vim 编辑模式                                               │
│  └── 基础插件系统                                               │
│                                                                  │
│  中期 (1 年):                                                    │
│  ├── Tree-sitter 语法高亮                                       │
│  ├── LSP 客户端                                                 │
│  ├── 活跃的插件生态                                             │
│  └── 跨平台支持                                                 │
│                                                                  │
│  长期:                                                           │
│  ├── 成为 Scheme 社区的首选编辑器                               │
│  ├── 独特的 Scheme-native 功能                                  │
│  │   ├── First-class continuation 支持                          │
│  │   ├── 宏定义编辑命令                                         │
│  │   └── 程序化编辑 (结构编辑)                                  │
│  └── 与其他 Scheme 项目集成 (Racket, Guile, etc.)               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 参考资料

### 项目

- [libvim](https://github.com/onivim/libvim) — Vim 引擎库
- [vim.wasm](https://github.com/rhysd/vim.wasm) — WebAssembly Vim
- [Edwin](https://www.gnu.org/software/mit-scheme/documentation/stable/mit-scheme-user/Edwin.html) — MIT Scheme 编辑器

### 文档

- [Chez Scheme FFI](https://www.scheme.com/csug8/foreign.html) — 外部函数接口
- [libvim.h API](https://github.com/onivim/libvim/blob/master/src/libvim.h) — API 定义
- [Neovim API](https://neovim.io/doc/user/api.html) — msgpack-rpc 参考

### 本 Fork

- `NvimServer/bin/build_libnvim.sh` — libnvim 构建脚本
- `Package.swift` — Swift 包定义
- `RxNeovim/Sources/RxNeovimApi.swift` — API 客户端示例
