# libvim API 功能参考

> libvim 是 Onivim 项目的 Vim 编辑引擎库，提供 Vim 的核心编辑功能作为可嵌入的 C 库。

## 目录

1. [概述](#1-概述)
2. [初始化](#2-初始化)
3. [Buffer 操作](#3-buffer-操作)
4. [Cursor 操作](#4-cursor-操作)
5. [用户输入](#5-用户输入)
6. [命令行](#6-命令行)
7. [Visual Mode](#7-visual-mode)
8. [搜索](#8-搜索)
9. [窗口](#9-窗口)
10. [选项](#10-选项)
11. [寄存器](#11-寄存器)
12. [Undo](#12-undo)
13. [宏](#13-宏)
14. [VimScript 执行](#14-vimscript-执行)
15. [模式查询](#15-模式查询)
16. [回调系统](#16-回调系统)
17. [不提供的功能](#17-不提供的功能)

---

## 1. 概述

### 1.1 什么是 libvim

libvim 是 Vim 的一个 fork，目标是提供**最小的 C API**来模拟 Vim 的模态编辑。它不包含任何用户界面，主要负责作为快速的 buffer 操作引擎。

### 1.2 设计理念

```
┌─────────────────────────────────────────────────────────────────┐
│                    libvim 设计理念                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "Vim 作为一个纯函数:                                            │
│   输入: (editor state, input)                                    │
│   输出: (new editor state)"                                      │
│                                                                  │
│  职责划分:                                                        │
│  ├── libvim 负责:                                                │
│  │   ├── Buffer 管理和操作                                       │
│  │   ├── 响应输入的 buffer 操作                                  │
│  │   ├── 解析和加载 VimL                                         │
│  │   └── 处理键映射                                              │
│  │                                                               │
│  └── 宿主应用负责:                                                │
│      ├── UI 渲染 (终端/GUI)                                      │
│      ├── 语法高亮                                                │
│      ├── 鼠标支持                                                │
│      ├── 补全                                                    │
│      └── 输入法 (IME)                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 主要用途

- **Onivim 2** — 主要使用者
- **WebAssembly 构建** — 浏览器中的 Vim 模式
- **原生应用** — 需要 Vim 绑定的应用程序
- **readline 替代** — 命令行编辑的基础

---

## 2. 初始化

```c
/**
 * vimInit
 * 
 * 使用任何其他方法之前必须调用。
 * 
 * @param argc 命令行参数数量
 * @param argv 命令行参数数组
 */
void vimInit(int argc, char **argv);
```

**使用示例:**

```c
int main(int argc, char **argv) {
    vimInit(argc, argv);
    // 现在可以使用其他 API
}
```

---

## 3. Buffer 操作

### 3.1 创建和打开

```c
/**
 * vimBufferOpen
 * 打开 buffer 并设为当前 buffer
 * 
 * @param ffname_arg 文件名
 * @param lnum       跳转到的行号
 * @param flags      标志位
 * @return           buffer 指针
 */
buf_T *vimBufferOpen(char_u *ffname_arg, linenr_T lnum, int flags);

/**
 * vimBufferLoad
 * 加载 buffer 但不切换当前 buffer
 */
buf_T *vimBufferLoad(char_u *ffname_arg, linenr_T lnum, int flags);

/**
 * vimBufferNew
 * 创建新 buffer
 */
buf_T *vimBufferNew(int flags);
```

### 3.2 查询

```c
/**
 * vimBufferGetById
 * 按 ID 获取 buffer
 */
buf_T *vimBufferGetById(int id);

/**
 * vimBufferGetCurrent
 * 获取当前 buffer
 */
buf_T *vimBufferGetCurrent(void);

/**
 * vimBufferGetFilename
 * 获取 buffer 的文件名
 */
char_u *vimBufferGetFilename(buf_T *buf);

/**
 * vimBufferGetFiletype
 * 获取 buffer 的文件类型
 */
char_u *vimBufferGetFiletype(buf_T *buf);

/**
 * vimBufferGetId
 * 获取 buffer 的 ID
 */
int vimBufferGetId(buf_T *buf);

/**
 * vimBufferGetLine
 * 获取指定行内容
 */
char_u *vimBufferGetLine(buf_T *buf, linenr_T lnum);

/**
 * vimBufferGetLineCount
 * 获取 buffer 的行数
 */
size_t vimBufferGetLineCount(buf_T *buf);

/**
 * vimBufferGetLastChangedTick
 * 获取最后修改的 tick
 */
long vimBufferGetLastChangedTick(buf_T *buf);
```

### 3.3 修改

```c
/**
 * vimBufferSetCurrent
 * 设置当前 buffer
 */
void vimBufferSetCurrent(buf_T *buf);

/**
 * vimBufferSetLines
 * 设置 buffer 中的行范围
 * 
 * start 参数是从 0 开始且包含的
 * end 参数是不包含的
 * 
 * 示例:
 *   vimBufferSetLines(buf, 0, 0, ["abc"], 1);  // 在第一行前插入 "abc"
 *   vimBufferSetLines(buf, 0, 1, ["abc"], 1);  // 将第 1 行设为 "abc"
 *   vimBufferSetLines(buf, 0, 2, ["abc"], 2);  // 第 1 行="abc", 第 2 行=空
 *   vimBufferSetLines(buf, 2, 2, ["abc"], 1);  // 在第 2 行后插入 "abc"
 */
void vimBufferSetLines(buf_T *buf, linenr_T start, linenr_T end, 
                       char_u **lines, int count);
```

### 3.4 状态

```c
/**
 * vimBufferGetModified
 * 获取 buffer 是否已修改
 */
int vimBufferGetModified(buf_T *buf);

/**
 * vimBufferGetModifiable / vimBufferSetModifiable
 * 获取/设置 buffer 是否可修改
 */
int vimBufferGetModifiable(buf_T *buf);
void vimBufferSetModifiable(buf_T *buf, int modifiable);

/**
 * vimBufferGetFileFormat / vimBufferSetFileFormat
 * 获取/设置文件格式
 */
int vimBufferGetFileFormat(buf_T *buf);
void vimBufferSetFileFormat(buf_T *buf, int fileformat);

/**
 * vimBufferGetReadOnly / vimBufferSetReadOnly
 * 获取/设置只读状态
 */
int vimBufferGetReadOnly(buf_T *buf);
void vimBufferSetReadOnly(buf_T *buf, int modifiable);

/**
 * vimBufferCheckIfChanged
 * 检查 buffer 内容是否在文件系统上被外部修改
 * @return 1 如果已修改 (并更新 buffer 内容)
 *         2 如果显示了消息
 *         0 否则
 */
int vimBufferCheckIfChanged(buf_T *buf);
```

### 3.5 Buffer 回调

```c
/**
 * vimSetBufferUpdateCallback
 * 设置 buffer 更新回调
 */
void vimSetBufferUpdateCallback(BufferUpdateCallback bufferUpdate);
```

---

## 4. Cursor 操作

### 4.1 获取位置

```c
/**
 * vimCursorGetColumn
 * 获取当前列号
 */
colnr_T vimCursorGetColumn(void);

/**
 * vimCursorGetLine
 * 获取当前行号
 */
linenr_T vimCursorGetLine(void);

/**
 * vimCursorGetPosition
 * 获取当前位置 (行 + 列)
 */
pos_T vimCursorGetPosition(void);

/**
 * vimCursorGetDesiredColumn
 * 获取期望的列 (用于上下移动时保持列位置)
 */
colnr_T vimCursorGetDesiredColumn(void);

/**
 * vimCursorGetColumnWant
 * 获取想要到达的列
 */
colnr_T vimCursorGetColumnWant(void);
```

### 4.2 设置位置

```c
/**
 * vimCursorSetPosition
 * 设置光标位置
 */
void vimCursorSetPosition(pos_T pos);

/**
 * vimCursorSetColumnWant
 * 设置想要到达的列
 */
void vimCursorSetColumnWant(colnr_T curswant);
```

### 4.3 光标回调

```c
/**
 * vimSetCursorAddCallback
 * 设置光标添加回调
 */
void vimSetCursorAddCallback(CursorAddCallback cursorAddCallback);

/**
 * vimSetCursorMoveScreenLineCallback
 * 设置屏幕行移动回调 (H, M, L 命令)
 */
void vimSetCursorMoveScreenLineCallback(
    CursorMoveScreenLineCallback cursorMoveScreenLineCallback);

/**
 * vimSetCursorMoveScreenPositionCallback
 * 设置屏幕位置移动回调 (gj, gk 命令)
 */
void vimSetCursorMoveScreenPositionCallback(
    CursorMoveScreenPositionCallback cursorMoveScreenPositionCallback);
```

---

## 5. 用户输入

### 5.1 核心输入函数

```c
/**
 * vimInput
 * 传递字符串给 vim 处理，不替换 term-codes
 * 这意味着像 "<LEFT>" 这样的字符串会被当作字面量处理
 * 此函数正确处理 Unicode 文本
 * 
 * @param input 输入字符串
 */
void vimInput(char_u *input);

/**
 * vimKey
 * 传递字符串并转义 termcodes
 * 像 "<LEFT>" 这样的字符串会被替换为适当的 term-code
 * 
 * @param key 键序列
 */
void vimKey(char_u *key);

/**
 * vimExecute
 * 执行命令，就像在命令行中输入一样
 * 
 * 示例: vimExecute("echo 'hello!'");
 */
void vimExecute(char_u *cmd);

/**
 * vimExecuteLines
 * 执行多行命令
 */
void vimExecuteLines(char_u **lines, int lineCount);
```

### 5.2 使用示例

```c
// 输入模式
vimInput("i");           // 进入插入模式
vimInput("hello");       // 输入文本
vimInput("\x1b");        // ESC，返回普通模式

// 使用 vimKey 处理特殊键
vimKey("<LEFT>");        // 左移
vimKey("<C-d>");         // Ctrl-D

// 执行 Ex 命令
vimExecute("w");         // 保存
vimExecute("%s/old/new/g"); // 全局替换
```

---

## 6. 命令行

```c
/**
 * vimCommandLineGetType
 * 获取命令行类型
 */
char_u vimCommandLineGetType(void);

/**
 * vimCommandLineGetText
 * 获取命令行文本
 */
char_u *vimCommandLineGetText(void);

/**
 * vimCommandLineGetPosition
 * 获取命令行光标位置
 */
int vimCommandLineGetPosition(void);

/**
 * vimCommandLineGetCompletions
 * 获取命令行补全列表
 */
void vimCommandLineGetCompletions(char_u ***completions, int *count);

/**
 * vimSetCustomCommandHandler
 * 设置自定义命令处理器
 */
void vimSetCustomCommandHandler(CustomCommandCallback customCommandHandler);
```

---

## 7. Visual Mode

```c
/**
 * vimVisualGetType
 * 获取 visual 类型
 */
int vimVisualGetType(void);

/**
 * vimVisualSetType
 * 设置 visual 类型
 */
void vimVisualSetType(int);

/**
 * vimVisualIsActive
 * 检查是否在 visual 模式
 */
int vimVisualIsActive(void);

/**
 * vimSelectIsActive
 * 检查是否在 select 模式
 */
int vimSelectIsActive(void);

/**
 * vimVisualGetRange
 * 获取 visual 选区范围
 * 如果在 visual 或 select 模式，返回当前范围
 * 否则返回上一次 visual 范围
 */
void vimVisualGetRange(pos_T *startPos, pos_T *endPos);

/**
 * vimVisualSetStart
 * 设置 visual 起点
 * 只在 visual 或 select 模式有效
 */
void vimVisualSetStart(pos_T startPos);
```

---

## 8. 搜索

```c
/**
 * vimSearchGetMatchingPair
 * 根据当前 buffer 和光标位置返回匹配的括号位置
 * 
 * @param initc 起始字符
 * @return      匹配位置，如果没有匹配则返回 NULL
 */
pos_T *vimSearchGetMatchingPair(int initc);

/**
 * vimSearchGetHighlights
 * 获取当前搜索的高亮
 */
void vimSearchGetHighlights(buf_T *buf, linenr_T start_lnum, linenr_T end_lnum,
                            int *num_highlights,
                            searchHighlight_T **highlights);

/**
 * vimSearchGetPattern
 * 获取当前搜索模式
 */
char_u *vimSearchGetPattern();

/**
 * vimSetStopSearchHighlightCallback
 * 设置停止搜索高亮回调
 */
void vimSetStopSearchHighlightCallback(VoidCallback callback);
```

---

## 9. 窗口

```c
/**
 * vimWindowGetWidth / vimWindowSetWidth
 * 获取/设置窗口宽度
 */
int vimWindowGetWidth(void);
void vimWindowSetWidth(int width);

/**
 * vimWindowGetHeight / vimWindowSetHeight
 * 获取/设置窗口高度
 */
int vimWindowGetHeight(void);
void vimWindowSetHeight(int height);

/**
 * vimWindowGetTopLine
 * 获取视口顶部行号
 */
int vimWindowGetTopLine(void);

/**
 * vimWindowGetLeftColumn
 * 获取视口左侧列号
 */
int vimWindowGetLeftColumn(void);

/**
 * vimWindowSetTopLeft
 * 设置视口位置
 */
void vimWindowSetTopLeft(int top, int left);

/**
 * vimSetWindowSplitCallback
 * 设置窗口分割回调
 */
void vimSetWindowSplitCallback(WindowSplitCallback callback);

/**
 * vimSetWindowMovementCallback
 * 设置窗口移动回调
 */
void vimSetWindowMovementCallback(WindowMovementCallback callback);
```

---

## 10. 选项

```c
/**
 * vimOptionSetTabSize / vimOptionGetTabSize
 * 设置/获取 Tab 大小
 */
void vimOptionSetTabSize(int tabSize);
int vimOptionGetTabSize(void);

/**
 * vimOptionSetInsertSpaces / vimOptionGetInsertSpaces
 * 设置/获取是否插入空格
 */
void vimOptionSetInsertSpaces(int insertSpaces);
int vimOptionGetInsertSpaces(void);

/**
 * vimSetOptionSetCallback
 * 设置选项设置回调
 */
void vimSetOptionSetCallback(OptionSetCallback callback);
```

---

## 11. 寄存器

```c
/**
 * vimRegisterGet
 * 获取寄存器内容
 * 
 * @param reg_name   寄存器名称
 * @param num_lines  输出: 行数
 * @param lines      输出: 行内容数组
 */
void vimRegisterGet(int reg_name, int *num_lines, char_u ***lines);
```

---

## 12. Undo

```c
/**
 * vimUndoSaveCursor
 * 保存光标状态
 */
int vimUndoSaveCursor(void);

/**
 * vimUndoSaveRegion
 * 保存区域状态
 */
int vimUndoSaveRegion(linenr_T start_lnum, linenr_T end_lnum);

/**
 * vimUndoSync
 * 创建同步点 (新的 undo 级别)
 * 停止添加到当前 undo 条目，开始新的条目
 * 
 * @param force 是否强制
 */
void vimUndoSync(int force);
```

---

## 13. 宏

```c
/**
 * vimMacroSetStartRecordCallback
 * 设置开始录制宏回调
 */
void vimMacroSetStartRecordCallback(MacroStartRecordCallback callback);

/**
 * vimMacroSetStopRecordCallback
 * 设置停止录制宏回调
 */
void vimMacroSetStopRecordCallback(MacroStopRecordCallback callback);
```

---

## 14. VimScript 执行

```c
/**
 * vimEval
 * 将字符串作为 VimScript 执行，并返回结果字符串
 * 调用者负责释放命令和结果
 * 
 * @param str VimScript 表达式
 * @return    执行结果字符串
 */
char_u *vimEval(char_u *str);
```

**使用示例:**

```c
// 执行 VimScript 表达式
char_u *result = vimEval("2 + 2");  // "4"
char_u *version = vimEval("v:version");  // 版本号
char_u *cwd = vimEval("getcwd()");  // 当前目录
```

---

## 15. 模式查询

```c
/**
 * vimGetMode
 * 获取当前模式
 */
int vimGetMode(void);

/**
 * vimGetSubMode
 * 获取子模式
 * 
 * 有些模态输入体验不是完整的模式，但仍然是模态输入状态
 * 例如: insert-literal (C-V, C-G), 带确认的搜索等
 */
subMode_T vimGetSubMode(void);

/**
 * vimGetPendingOperator
 * 获取待处理的操作符
 */
int vimGetPendingOperator(pendingOp_T *pendingOp);
```

---

## 16. 回调系统

回调是 libvim 与宿主应用通信的核心机制。

### 16.1 完整回调列表

| 回调函数 | 触发时机 |
|----------|----------|
| `vimSetBufferUpdateCallback` | Buffer 内容变化 |
| `vimSetAutoCommandCallback` | Autocmd 触发 |
| `vimSetMessageCallback` | 显示消息 |
| `vimSetClearCallback` | 清除消息/各种实体 |
| `vimSetOutputCallback` | `:!cmd` 输出 |
| `vimSetScrollCallback` | 滚动请求 (C-Y, zz 等) |
| `vimSetCursorMoveScreenLineCallback` | H/M/L 命令 |
| `vimSetCursorMoveScreenPositionCallback` | gj/gk 命令 |
| `vimSetYankCallback` | Yank 操作 |
| `vimSetQuitCallback` | `:q`, `:qa`, `:q!` 命令 |
| `vimSetFormatCallback` | 格式化请求 |
| `vimSetGotoCallback` | 跳转请求 |
| `vimSetTabPageCallback` | 标签页操作 |
| `vimSetDirectoryChangedCallback` | 目录变化 |
| `vimSetOptionSetCallback` | 选项设置 |
| `vimSetToggleCommentsCallback` | 注释切换 |
| `vimSetWindowSplitCallback` | 窗口分割 |
| `vimSetWindowMovementCallback` | 窗口移动 |
| `vimSetTerminalCallback` | 终端操作 |
| `vimSetClipboardGetCallback` | 剪贴板获取 |
| `vimColorSchemeSetChangedCallback` | 配色变化 |
| `vimColorSchemeSetCompletionCallback` | 配色补全 |
| `vimSetInputMapCallback` | 映射设置 |
| `vimSetInputUnmapCallback` | 取消映射 |
| `vimSetFileWriteFailureCallback` | 文件写入失败 |
| `vimSetCustomCommandHandler` | 自定义命令 |
| `vimSetAutoIndentCallback` | 自动缩进 |
| `vimSetCursorAddCallback` | 光标添加 |
| `vimSetUnhandledEscapeCallback` | Normal 模式未处理的 ESC |
| `vimSetStopSearchHighlightCallback` | 停止搜索高亮 |
| `vimSetDisplayIntroCallback` | 显示 intro |
| `vimSetDisplayVersionCallback` | 显示版本 |
| `vimSetFunctionGetCharCallback` | 获取字符 |
| `vimSetDirectoryChangedCallback` | 目录变化 |

### 16.2 关键回调详解

#### Buffer 更新回调

```c
// 当 buffer 内容变化时调用
void vimSetBufferUpdateCallback(BufferUpdateCallback bufferUpdate);

// 回调签名示例
typedef void (*BufferUpdateCallback)(buf_T *buf, 
                                       linenr_T linenr, 
                                       linenr_T nlines, 
                                       char_u **lines);
```

#### 消息回调

```c
// 当 Vim 需要显示消息时调用
void vimSetMessageCallback(MessageCallback messageCallback);

// 回调签名示例
typedef void (*MessageCallback)(char_u *message, int priority);
```

#### 退出回调

```c
// 当 :q, :qa, :q! 被调用时
void vimSetQuitCallback(QuitCallback callback);

// 回调签名
// buffer: 请求退出的 buffer
// force: 是否强制 (如 q!)
typedef void (*QuitCallback)(buf_T *buffer, int force);
```

#### 滚动回调

```c
// 当窗口应该滚动时 (C-Y, zz 等)
void vimSetScrollCallback(ScrollCallback callback);

// 回调签名
typedef void (*ScrollCallback)(int direction, int count);
```

---

## 17. 不提供的功能

libvim **明确不负责**以下功能，这些需要宿主应用实现：

```
┌─────────────────────────────────────────────────────────────────┐
│                 libvim 不提供的功能                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ UI 渲染                                                      │
│     ├── 终端 UI                                                  │
│     ├── GUI 渲染                                                 │
│     └── 任何形式的显示                                           │
│                                                                  │
│  ❌ 交互功能                                                      │
│     ├── 鼠标支持                                                 │
│     ├── 输入法 (IME)                                             │
│     └── 补全 UI                                                  │
│                                                                  │
│  ❌ 高级编辑功能                                                  │
│     ├── 语法高亮                                                 │
│     ├── 拼写检查                                                 │
│     └── 终端模拟                                                 │
│                                                                  │
│  这些功能由 libvim 的宿主应用负责实现。                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 18. 使用示例

### 18.1 基本使用流程

```c
#include "libvim.h"

int main(int argc, char **argv) {
    // 1. 初始化
    vimInit(argc, argv);
    
    // 2. 设置回调
    vimSetBufferUpdateCallback(my_buffer_update_handler);
    vimSetMessageCallback(my_message_handler);
    
    // 3. 打开文件
    buf_T *buf = vimBufferOpen("test.txt", 1, 0);
    
    // 4. 发送输入
    vimInput("i");           // 进入插入模式
    vimInput("Hello World"); // 输入文本
    vimKey("<Esc>");         // 返回普通模式
    
    // 5. 执行命令
    vimExecute("w");         // 保存
    
    // 6. 获取 buffer 内容
    size_t line_count = vimBufferGetLineCount(buf);
    for (size_t i = 1; i <= line_count; i++) {
        char_u *line = vimBufferGetLine(buf, i);
        printf("Line %zu: %s\n", i, line);
    }
    
    return 0;
}
```

### 18.2 Scheme FFI 绑定示例

```scheme
;; vim-ffi.ss

;; 加载共享库
(load-shared-object "./libvim.so")

;; 基础绑定
(define vim-init
  (foreign-procedure "vimInit" (int ptr) void))

(define vim-input
  (foreign-procedure "vimInput" (string) void))

(define vim-key
  (foreign-procedure "vimKey" (string) void))

(define vim-execute
  (foreign-procedure "vimExecute" (string) void))

(define vim-buffer-open
  (foreign-procedure "vimBufferOpen" (string int int) ptr))

(define vim-buffer-get-line
  (foreign-procedure "vimBufferGetLine" (ptr int) string))

(define vim-buffer-get-line-count
  (foreign-procedure "vimBufferGetLineCount" (ptr) size_t))

;; 高级 API
(define (vim-insert text)
  (vim-input "i")
  (vim-input text)
  (vim-key "<Esc>"))

(define (vim-get-buffer-contents buf)
  (let ((count (vim-buffer-get-line-count buf)))
    (let loop ((i 1) (lines '()))
      (if (> i count)
          (reverse lines)
          (loop (+ i 1)
                (cons (vim-buffer-get-line buf i) lines))))))

;; 使用
(vim-init 0 #f)
(define buf (vim-buffer-open "test.scm" 1 0))
(vim-insert "(define hello \"world\")")
(vim-execute "w")
```

---

## 参考资料

- [libvim GitHub](https://github.com/onivim/libvim)
- [libvim.h API 头文件](https://github.com/onivim/libvim/blob/master/src/libvim.h)
- [Onivim 2](https://v2.onivim.io) — 主要使用者
