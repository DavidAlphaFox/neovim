# 删除 Lua 支持的影响分析

> 本文档分析如果删除 Lua 支持，Neovim 将失去哪些功能。

## 目录

1. [结论](#1-结论)
2. [完全丢失的功能 (纯 Lua 实现)](#2-完全丢失的功能-纯-lua-实现)
3. [C 代码中依赖 Lua 的功能](#3-c-代码中依赖-lua-的功能)
4. [会失效的命令和函数](#4-会失效的命令和函数)
5. [删除 Vimscript vs 删除 Lua 对比](#5-删除-vimscript-vs-删除-lua-对比)
6. [Lua Runtime 文件清单](#6-lua-runtime-文件清单)
7. [C-Lua 桥接点](#7-c-lua-桥接点)
8. [架构总结](#8-架构总结)

---

## 1. 结论

```
┌─────────────────────────────────────────────────────────────────┐
│                     哪个更"致命"？                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  删除 Vimscript:                                                 │
│  - 失去的是"用户体验层" (syntax, indent, ftplugin)              │
│  - 可以用 Lua/Tree-sitter 完全替代                               │
│  - 老插件生态会丢失                                              │
│  - Neovim 4.0+ 的方向                                            │
│                                                                  │
│  删除 Lua:                                                       │
│  - 失去的是"现代核心功能" (LSP, Tree-sitter, Diagnostic)         │
│  - 无法用 Vimscript 替代 (这些功能是 Lua 实现的)                 │
│  - 现代插件生态会全部丢失                                        │
│  - 回退到"传统 Vim + 异步支持"                                   │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  结论: 删除 Lua 比 删除 Vimscript 影响更大                       │
│        因为现代 Neovim 的核心差异化功能都是 Lua 实现的           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 完全丢失的功能 (纯 Lua 实现)

### 2.1 内置 LSP 客户端 — 完全依赖 Lua

| 文件 | 行数 | 功能 |
|------|------|------|
| `vim/lsp.lua` | 1887 | LSP 客户端主入口 |
| `vim/lsp/buf.lua` | - | Buffer 级 LSP 操作 |
| `vim/lsp/util.lua` | - | LSP 工具函数 |
| `vim/lsp/rpc.lua` | - | LSP RPC 通信 |
| `vim/lsp/protocol.lua` | - | LSP 协议定义 |
| `vim/lsp/handlers.lua` | - | LSP 事件处理器 |
| `vim/lsp/diagnostic.lua` | - | LSP 诊断集成 |
| `vim/lsp/codelens.lua` | - | Code Lens 支持 |
| `vim/lsp/tagfunc.lua` | - | LSP tagfunc |
| `vim/lsp/sync.lua` | - | 同步工具 |
| `vim/lsp/_snippet.lua` | - | 片段支持 |

**丢失的功能：**
- `vim.lsp.start()`, `vim.lsp.buf.*`, `vim.lsp.clients`
- 跳转到定义/声明/实现
- 查找引用
- 悬停文档
- 代码补全
- 代码操作
- 重命名
- 签名帮助
- Code Lens
- 文档高亮
- 工作区符号

**无 Vimscript 替代方案。**

### 2.2 Tree-sitter 集成 — 完全依赖 Lua

| 文件 | 功能 |
|------|------|
| `vim/treesitter.lua` | 主 API |
| `vim/treesitter/query.lua` | 查询语言 |
| `vim/treesitter/highlighter.lua` | 语法高亮 |
| `vim/treesitter/languagetree.lua` | 语言树解析 |
| `vim/treesitter/language.lua` | 语言运行时 |
| `vim/treesitter/health.lua` | 健康检查 |

**丢失的功能：**
- `vim.treesitter.get_parser()`
- `vim.treesitter.query.*`
- 基于 Tree-sitter 的语法高亮
- 语言树解析和注入
- 增量解析

**无 Vimscript 替代方案。**

### 2.3 诊断系统 — 完全依赖 Lua

| 文件 | 行数 | 功能 |
|------|------|------|
| `vim/diagnostic.lua` | 1625 | 诊断 API 和显示 |

**丢失的功能：**
- `vim.diagnostic.get()`, `vim.diagnostic.set()`
- `vim.diagnostic.open_float()`, `vim.diagnostic.goto_next/prev()`
- 诊断符号、虚拟文本、下划线渲染
- 与 quickfix/loclist 集成
- 严重级别过滤

**无 Vimscript 替代方案。**

### 2.4 现代 UI API — Lua 专属

| 文件 | 功能 |
|------|------|
| `vim/ui.lua` | `vim.ui.select()`, `vim.ui.input()` |

**丢失的功能：**
- `vim.ui.select()` — 现代选择对话框
- `vim.ui.input()` — 现代输入对话框
- UI 可扩展性 (插件可覆盖实现，如 Telescope, fzf)

**无 Vimscript 替代方案。**

### 2.5 扩展高亮 API — Lua 专属

| 文件 | 行数 | 功能 |
|------|------|------|
| `vim/highlight.lua` | 139 | 高亮工具函数 |

**丢失的功能：**
- `vim.highlight.range()` — 范围高亮
- `vim.highlight.on_yank()` — Yank 高亮
- 优先级高亮系统

### 2.6 现代 Keymap API — Lua 增强

| 文件 | 行数 | 功能 |
|------|------|------|
| `vim/keymap.lua` | 145 | Lua keymap API |

**丢失的功能：**
- `vim.keymap.set()` 支持 Lua 函数作为 RHS
- 表达式映射中使用 Lua 函数
- 更优雅的 Buffer 局部映射 API

**Vimscript 可通过 `luaeval()` 实现部分功能，但不够优雅。**

### 2.7 其他 Lua 专属功能

| 文件 | 功能 |
|------|------|
| `vim/F.lua` | 函数式编程工具 (`if_nil`, `npcall`) |
| `vim/uri.lua` | URI 处理 |
| `vim/filetype.lua` | 现代文件类型检测 |
| `vim/inspect.lua` | 对象检查工具 |

---

## 3. C 代码中依赖 Lua 的功能

### 3.1 功能与文件对照表

| 功能 | 文件 | 调用点 | 影响 |
|------|------|--------|------|
| **Autocmd Lua 回调** | `autocmd.c:2065` | `nlua_call_ref` | 用 Lua 函数注册的 autocmd 不执行 |
| **Buffer 更新回调** | `buffer_updates.c:171,285,335,370` | `nlua_call_ref` | `nvim_buf_attach()` 的 on_lines/on_bytes/on_changedtick |
| **键盘映射 Lua 回调** | `getchar.c:4063,4730` | `nlua_call_ref` | 映射到 Lua 函数的键不工作 |
| **装饰提供者** | `decoration_provider.c:26` | `nlua_call_ref` | extmark 的装饰回调不工作 |
| **高亮定义** | `highlight.c:194` | `nlua_call_ref` | 用 Lua 定义的高亮组不工作 |
| **用户自定义补全** | `ex_getln.c:5400` | `nlua_call_user_expand_func` | Lua 补全函数不工作 |
| **v:lua 调用** | `eval/userfunc.c:1525` | `nlua_typval_call` | Vimscript 调用 Lua 不工作 |
| **luaeval()** | `eval/funcs.c:5726` | `nlua_typval_eval` | Vimscript 中执行 Lua 不工作 |
| **窗口配置回调** | `api/window.c:463` | `nlua_call_ref` | `nvim_win_set_config` 的回调不工作 |
| **Buffer API 回调** | `api/buffer.c:1369` | `nlua_call_ref` | Buffer 相关 API 回调不工作 |
| **终端输入回调** | `api/vim.c:1170` | `nlua_call_ref` | 终端输入回调不工作 |

### 3.2 核心 Lua 子系统

| 文件 | 功能 |
|------|------|
| `lua/executor.c` | 核心 Lua 执行引擎：`nlua_call`, `nlua_pcall`, `nlua_call_ref`, `nlua_typval_eval`, `vim.schedule` |
| `lua/converter.c` | Lua ↔ Vim 类型转换 |
| `lua/stdlib.c` | 正则、字符串工具 (C 绑定) |
| `lua/spell.c` | 拼写检查模块 |
| `lua/treesitter.c` | Tree-sitter 解析器集成 |
| `lua/xdiff.c` | Diff 算法绑定 |

---

## 4. 会失效的命令和函数

### 4.1 Ex 命令

```vim
:lua ...          " 执行 Lua 代码 — 失效
:luarequire ...   " require Lua 模块 — 失效
:luafile ...      " 加载 Lua 文件 — 失效
```

### 4.2 Vimscript 函数

```vim
luaeval('expr', {args})  " 在 Vimscript 中执行 Lua — 失效
v:lua.func_name()        " 调用 Lua 全局函数 — 失效
```

### 4.3 选项 (如果设置为 Lua 函数)

```vim
set operatorfunc=v:lua.MyFunc    " 失效
set completefunc=v:lua.MyComplete " 失效
set omnifunc=v:lua.MyOmni        " 失效
```

---

## 5. 删除 Vimscript vs 删除 Lua 对比

### 5.1 功能对比

| 特性 | 删除 Vimscript | 删除 Lua |
|------|---------------|----------|
| 核心编辑 | ✅ 正常 | ✅ 正常 |
| Ex 命令 | ✅ 大部分正常 | ✅ 正常 |
| 语法高亮 | ⚠️ 需要 Tree-sitter/Lua | ⚠️ 仅能用 syntax/*.vim |
| 文件类型检测 | ⚠️ 需要 filetype.lua | ⚠️ 仅能用 filetype.vim |
| LSP | ✅ 有 (Lua 实现) | ❌ **完全丢失** |
| Tree-sitter | ✅ 有 (Lua API) | ❌ **高级 API 丢失** |
| 诊断系统 | ✅ 有 (Lua 实现) | ❌ **完全丢失** |
| 现代插件 | ✅ Lua 插件正常 | ❌ **几乎全部丢失** |
| 老插件 | ❌ 丢失 | ✅ Vimscript 插件正常 |
| 配置文件 | init.lua | init.vim |

### 5.2 恢复可能性对比

| | 删除 Vimscript | 删除 Lua |
|---|---------------|----------|
| **能否恢复功能** | ✅ 用 Lua 重写 | ❌ 无替代 (需用 C 重写) |
| **对现代插件的影响** | 低 (大多用 Lua) | 致命 (几乎全用 Lua) |
| **对传统用户的影响** | 高 (老配置不工作) | 低 (init.vim 正常) |
| **与 Vim 的区别** | 变大 (更依赖 Lua) | 变小 (退回传统 Vim) |

---

## 6. Lua Runtime 文件清单

### 6.1 核心文件

```
runtime/lua/vim/
├── _init_packages.lua   # 包加载器初始化
├── _editor.lua          # 编辑器 API (vim.fn, vim.cmd, vim.g 等)
├── _meta.lua            # vim.opt, vim.bo, vim.wo 等元数据
├── shared.lua           # 纯 Lua 工具函数
├── F.lua                # 函数式编程工具
├── inspect.lua          # 对象检查
├── uri.lua              # URI 处理
├── compat.lua           # 兼容层
├── filetype.lua         # 文件类型检测
├── keymap.lua           # Keymap API
├── ui.lua               # UI API (select, input)
├── highlight.lua        # 高亮工具
├── diagnostic.lua       # 诊断系统 (1625 行)
└── treesitter.lua       # Tree-sitter 主 API
```

### 6.2 LSP 子目录

```
runtime/lua/vim/lsp/
├── lsp.lua              # 主 LSP 实现 (1887 行)
├── buf.lua              # Buffer 操作
├── util.lua             # 工具函数
├── rpc.lua              # RPC 通信
├── protocol.lua         # 协议定义
├── handlers.lua         # 事件处理器
├── diagnostic.lua       # LSP 诊断
├── codelens.lua         # Code Lens
├── tagfunc.lua          # tagfunc
├── sync.lua             # 同步工具
├── log.lua              # 日志
├── health.lua           # 健康检查
└── _snippet.lua         # 片段支持
```

### 6.3 Tree-sitter 子目录

```
runtime/lua/vim/treesitter/
├── treesitter.lua       # 主 API (117 行)
├── query.lua            # 查询语言
├── highlighter.lua      # 语法高亮
├── languagetree.lua     # 语言树
├── language.lua         # 语言运行时
└── health.lua           # 健康检查
```

---

## 7. C-Lua 桥接点

### 7.1 C 调用 Lua 的入口

| 函数 | 文件 | 用途 |
|------|------|------|
| `nlua_call` | `lua/executor.c:920` | 调用 Vim 函数 |
| `nlua_pcall` | `lua/executor.c:123` | 受保护的 Lua 调用 |
| `nlua_call_ref` | `lua/executor.c:1353` | 通过引用调用 Lua 函数 |
| `nlua_typval_eval` | `lua/executor.c:1144` | 执行 Lua 表达式 |
| `nlua_typval_call` | `lua/executor.c:1166` | v:lua 调用 |
| `nlua_schedule` | `lua/executor.c:318` | 调度到主线程 |

### 7.2 类型转换

| 函数 | 文件 | 用途 |
|------|------|------|
| `nlua_push_typval` | `lua/converter.c` | Vim → Lua 类型转换 |
| `nlua_pop_typval` | `lua/converter.c` | Lua → Vim 类型转换 |
| `nlua_push_Object` | `lua/converter.c` | API Object → Lua |
| `nlua_pop_Object` | `lua/converter.c` | Lua → API Object |

### 7.3 内置模块

| 模块 | 文件 | 功能 |
|------|------|------|
| `vim.regex` | `lua/stdlib.c` | Vim 正则表达式 |
| `vim.spell` | `lua/spell.c` | 拼写检查 |
| `vim.diff` | `lua/xdiff.c` | Diff 算法 |
| `vim.treesitter.*` | `lua/treesitter.c` | Tree-sitter C API |

---

## 8. 架构总结

### 8.1 依赖关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                      现代插件生态                                │
│  nvim-lspconfig, nvim-treesitter, Telescope, nvim-cmp, etc.    │
│                           ↓ 全部依赖 Lua                        │
├─────────────────────────────────────────────────────────────────┤
│                      Lua 标准库 (runtime/lua/vim/)              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ vim.lsp  │ │vim.treesitter│ │vim.diagnostic│ │ vim.ui/keymap │   │
│  │ (1887行) │ │  (~1000行)  │ │  (1625行)  │ │    (~250行)    │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
│                           ↓ 调用                                │
├─────────────────────────────────────────────────────────────────┤
│                      C-Lua 桥接层 (src/nvim/lua/)               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ executor │ │converter │ │ stdlib   │ │ treesitter/xdiff │   │
│  │  (1915行)│ │ (1326行) │ │  (541行) │ │   (~1000行)      │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
│                           ↓ 调用                                │
├─────────────────────────────────────────────────────────────────┤
│                      核心 C 实现 (src/nvim/)                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ normal.c │ │  ops.c   │ │ buffer.c │ │   autocmd.c      │   │
│  │ edit.c   │ │ search.c │ │ window.c │ │   buffer_updates │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 功能依赖矩阵

| 功能层 | Vimscript 可替代 | Lua 可替代 | 必须用 C |
|--------|-----------------|-----------|----------|
| 核心编辑 | - | - | ✅ |
| Ex 命令 | ✅ | ✅ | - |
| 语法高亮 | ✅ (syntax/*.vim) | ✅ (Tree-sitter) | - |
| 文件类型检测 | ✅ (filetype.vim) | ✅ (filetype.lua) | - |
| LSP 客户端 | ❌ | ✅ | ❌ |
| Tree-sitter | ❌ | ✅ | 部分 |
| 诊断系统 | ❌ | ✅ | ❌ |
| 现代插件 | ❌ | ✅ | ❌ |
| 老插件 | ✅ | ❌ | - |

### 8.3 最终结论

```
┌─────────────────────────────────────────────────────────────────┐
│                    删除 Lua 的影响                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ 仍然可用:                                                    │
│  - 核心编辑功能 (d, y, p, i, a, x, 等)                          │
│  - 窗口/Buffer 管理                                              │
│  - Ex 命令                                                       │
│  - Vimscript 语法高亮 (syntax/*.vim)                            │
│  - Vimscript 插件                                                │
│  - init.vim 配置                                                 │
│                                                                  │
│  ❌ 完全丢失:                                                     │
│  - 内置 LSP 客户端                                               │
│  - Tree-sitter 高级 API                                          │
│  - 诊断系统 (vim.diagnostic)                                     │
│  - vim.ui.select/input                                           │
│  - vim.keymap (Lua 函数映射)                                     │
│  - 现代插件生态 (95%+ 依赖 Lua)                                  │
│  - init.lua 配置                                                 │
│  - luaeval() / v:lua                                             │
│  - Lua 回调的 autocmd                                            │
│  - Buffer 更新回调 (on_lines 等)                                 │
│  - 装饰提供者 (decoration providers)                             │
│                                                                  │
│  ⚠️ 结论: 删除 Lua = 回退到"传统 Vim + 异步"                     │
│          失去 Neovim 的核心差异化功能                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 参考资料

- `:help lua` — Lua 集成文档
- `:help lsp` — LSP 文档
- `:help treesitter` — Tree-sitter 文档
- `runtime/lua/vim/` — Lua 标准库源码
- `src/nvim/lua/` — C-Lua 桥接层源码
