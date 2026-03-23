# Neovim 脚本系统架构分析

> 本文档总结 Neovim 的 Lua 和 Vimscript 双脚本系统架构，以及删除任一脚本支持的影响对比。

## 文档索引

| 文档 | 内容 |
|------|------|
| [lua-embedding-architecture.md](lua-embedding-architecture.md) | Lua 嵌入架构详解 |
| [vimscript-dependency-analysis.md](vimscript-dependency-analysis.md) | 删除 Vimscript 的影响 |
| [lua-dependency-analysis.md](lua-dependency-analysis.md) | 删除 Lua 的影响 |
| [chez-scheme-feasibility.md](chez-scheme-feasibility.md) | Chez Scheme 替代可行性 |
| [reverse-architecture-vim-in-scheme.md](reverse-architecture-vim-in-scheme.md) | 反向架构：Vim 嵌入 Chez Scheme |
| [libvim-api-reference.md](libvim-api-reference.md) | libvim API 参考 |
| [tui-options-for-scheme-editor.md](tui-options-for-scheme-editor.md) | **Chez Scheme + libvim 的 TUI 选项** |

---

## 1. 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                         Neovim 架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    用户脚本层                             │    │
│  │  ┌─────────────────┐     ┌─────────────────┐            │    │
│  │  │   init.lua      │     │   init.vim      │            │    │
│  │  │   (现代配置)     │     │   (传统配置)     │            │    │
│  │  └─────────────────┘     └─────────────────┘            │    │
│  │           │                      │                       │    │
│  │           ▼                      ▼                       │    │
│  │  ┌─────────────────┐     ┌─────────────────┐            │    │
│  │  │  Lua Runtime    │     │ Vimscript Runtime│            │    │
│  │  │  vim/lsp        │     │  syntax/*.vim   │            │    │
│  │  │  vim/treesitter │     │  indent/*.vim   │            │    │
│  │  │  vim.diagnostic │     │  ftplugin/*.vim │            │    │
│  │  │  vim.keymap     │     │  plugin/*.vim   │            │    │
│  │  └─────────────────┘     └─────────────────┘            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    脚本引擎层                             │    │
│  │  ┌─────────────────┐     ┌─────────────────┐            │    │
│  │  │   Lua VM        │     │ Vimscript Eval  │            │    │
│  │  │  (lua/executor) │◄───►│   (eval/)       │            │    │
│  │  └─────────────────┘     └─────────────────┘            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                      │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    核心 C 层                              │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐    │    │
│  │  │normal.c │ │ ops.c   │ │buffer.c │ │  window.c   │    │    │
│  │  │edit.c   │ │search.c │ │ undo.c  │ │ ex_docmd.c  │    │    │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────────┘    │    │
│  │              完全独立于脚本系统                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心发现

### 2.1 删除 Vimscript vs 删除 Lua

| 特性 | 删除 Vimscript | 删除 Lua |
|------|---------------|----------|
| **核心编辑** | ✅ 正常 | ✅ 正常 |
| **Ex 命令** | ✅ 大部分正常 | ✅ 正常 |
| **LSP 客户端** | ✅ 有 (Lua 实现) | ❌ **完全丢失** |
| **Tree-sitter** | ✅ 有 (Lua API) | ❌ **高级 API 丢失** |
| **诊断系统** | ✅ 有 (Lua 实现) | ❌ **完全丢失** |
| **现代 UI API** | ✅ 有 (Lua 实现) | ❌ **完全丢失** |
| **现代插件** | ✅ Lua 插件正常 | ❌ **几乎全部丢失** |
| **传统插件** | ❌ 丢失 | ✅ Vimscript 插件正常 |
| **语法高亮** | ⚠️ 需要 Tree-sitter | ⚠️ 仅能用 syntax/*.vim |
| **文件类型检测** | ⚠️ 需要 filetype.lua | ⚠️ 仅能用 filetype.vim |

### 2.2 恢复可能性

| | 删除 Vimscript | 删除 Lua |
|---|---------------|----------|
| **能否恢复功能** | ✅ 用 Lua 重写 | ❌ 无替代 (需用 C 重写) |
| **对现代插件的影响** | 低 (大多用 Lua) | 致命 (几乎全用 Lua) |
| **对传统用户的影响** | 高 (老配置不工作) | 低 (init.vim 正常) |
| **与 Vim 的区别** | 变大 (更依赖 Lua) | 变小 (退回传统 Vim) |

### 2.3 结论

```
┌─────────────────────────────────────────────────────────────────┐
│  删除 Lua 比 删除 Vimscript 影响更大                            │
│                                                                  │
│  原因：                                                          │
│  - LSP、Tree-sitter、诊断系统 完全是 Lua 实现                   │
│  - 这些功能无 Vimscript 替代方案                                 │
│  - 现代插件生态 95%+ 依赖 Lua                                    │
│                                                                  │
│  删除 Vimscript:                                                 │
│  → 失去"用户体验层" (syntax, indent, ftplugin)                  │
│  → 可用 Lua/Tree-sitter 完全替代                                 │
│  → 是 Neovim 的演进方向                                          │
│                                                                  │
│  删除 Lua:                                                       │
│  → 失去"现代核心功能" (LSP, Tree-sitter, Diagnostic)            │
│  → 无法用 Vimscript 替代                                         │
│  → 回退到"传统 Vim + 异步支持"                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 功能依赖矩阵

### 3.1 Lua 独占功能 (无 Vimscript 替代)

| 功能 | 实现文件 | 行数 |
|------|----------|------|
| **LSP 客户端** | `vim/lsp/*.lua` | ~4000+ |
| **Tree-sitter API** | `vim/treesitter/*.lua` | ~1000+ |
| **诊断系统** | `vim/diagnostic.lua` | 1625 |
| **UI API** | `vim/ui.lua` | 99 |
| **Keymap API** | `vim/keymap.lua` | 145 |
| **Highlight API** | `vim/highlight.lua` | 139 |

### 3.2 Vimscript 独占功能 (无 Lua 替代)

| 功能 | 实现文件 |
|------|----------|
| **传统语法高亮** | `syntax/*.vim` (600+ 文件) |
| **传统缩进规则** | `indent/*.vim` (200+ 文件) |
| **传统文件类型插件** | `ftplugin/*.vim` (200+ 文件) |
| **传统插件** | `plugin/*.vim` |

### 3.3 C 核心功能 (独立于脚本)

| 功能 | 实现文件 | 说明 |
|------|----------|------|
| **Normal 模式** | `normal.c` | d, y, p, x, c, r 等 |
| **Insert 模式** | `edit.c` | i, a, o, O 等 |
| **操作符** | `ops.c` | 删除, yank, put |
| **Buffer 管理** | `buffer.c` | :bnext, :bprev, :bd |
| **窗口管理** | `window.c` | :split, :vsplit, :close |
| **搜索** | `search.c` | /, ?, n, N, *, # |
| **撤销** | `undo.c` | u, Ctrl-R |
| **Ex 命令解析** | `ex_docmd.c` | 所有 : 命令 |

---

## 4. 启动流程

```
main.c:main()
    │
    ├── early_init()           # 早期初始化
    │
    ├── nlua_init()            # 初始化 Lua VM ◀─── 关键点
    │   └── luaL_newstate()
    │   └── nlua_state_init()
    │       ├── 创建 vim 表
    │       ├── nlua_add_api_functions() → vim.api
    │       ├── nlua_init_types() → vim.types
    │       └── nlua_init_packages() → 加载 runtime/lua/vim/
    │
    ├── [如果 -u NONE 则跳过以下]
    │
    ├── filetype_plugin_enable()   # 加载 ftplugin.vim + indent.vim
    ├── source_startup_scripts()   # 加载 init.lua 或 init.vim
    ├── filetype_maybe_enable()    # 加载 filetype.lua + filetype.vim
    ├── syn_maybe_enable()         # 加载语法高亮
    └── load_plugins()             # 加载 plugin/*.vim
```

---

## 5. C-Lua 互操作

### 5.1 C 调用 Lua

| 调用点 | 文件 | 功能 |
|--------|------|------|
| `nlua_call_ref` | `autocmd.c` | Autocmd Lua 回调 |
| `nlua_call_ref` | `buffer_updates.c` | Buffer 更新回调 (on_lines 等) |
| `nlua_call_ref` | `getchar.c` | 键盘映射 Lua 回调 |
| `nlua_call_ref` | `decoration_provider.c` | 装饰提供者 |
| `nlua_typval_eval` | `eval/funcs.c` | luaeval() 函数 |
| `nlua_typval_call` | `eval/userfunc.c` | v:lua 调用 |

### 5.2 Lua 调用 C

| API | 实现 | 功能 |
|-----|------|------|
| `vim.api.*` | `api/*.c` | 核心 API |
| `vim.loop` | `luv` | libuv 绑定 |
| `vim.mpack` | `lmpack` | MessagePack |
| `vim.json` | `lua_cjson` | JSON |
| `vim.diff` | `lua/xdiff.c` | Diff |
| `vim.regex` | `lua/stdlib.c` | 正则 |

---

## 6. Neovim 演进方向

### 6.1 从 Vimscript 到 Lua 的迁移

| 功能 | 旧方案 (Vimscript) | 新方案 (Lua/Tree-sitter) | 状态 |
|------|-------------------|-------------------------|------|
| 文件类型检测 | `filetype.vim` | `filetype.lua` | ✅ 完成 |
| 语法高亮 | `syntax/*.vim` | Tree-sitter | ⏳ 进行中 |
| 缩进规则 | `indent/*.vim` | Tree-sitter queries | ⏳ 进行中 |
| 配置文件 | `init.vim` | `init.lua` | ✅ 完成 |
| 插件 | Vimscript | Lua (via vim.api) | ✅ 完成 |
| LSP | 无 | 内置 Lua 实现 | ✅ 完成 |

### 6.2 未来展望

```
┌─────────────────────────────────────────────────────────────────┐
│                    Neovim 未来架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  完全 Lua 化:                                                    │
│  ├── init.lua 作为唯一配置文件                                  │
│  ├── Tree-sitter 替代所有 syntax/*.vim                          │
│  ├── Tree-sitter queries 替代所有 indent/*.vim                  │
│  ├── 所有新功能用 Lua 实现                                      │
│  └── Vimscript 仅保留向后兼容                                   │
│                                                                  │
│  Vimscript 将成为:                                               │
│  ├── 遗留兼容层                                                  │
│  ├── 仅支持传统插件                                              │
│  └── 不再添加新特性                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 验证方法

### 7.1 无 Vimscript 测试

```bash
# 启动无 Vimscript 配置的 Neovim
nvim -u NONE

# 功能测试:
# - 编辑: i, a, o, d, y, p, x, c, r ✓
# - 窗口: :split, :vsplit, :close ✓
# - Buffer: :bnext, :bprev, :bd ✓
# - 搜索: /, ?, n, N ✓
# - 撤销: u, Ctrl-R ✓
# - LSP: ✗ (需要 Lua runtime)
# - Tree-sitter: ✗ (需要 Lua runtime)
```

### 7.2 无 Lua 测试

```bash
# 无法完全禁用 Lua (Lua VM 在启动时初始化)
# 但可以测试 Vimscript 配置:
nvim -u NORC  # 使用 init.vim
```

---

## 参考资料

- `:help lua` — Lua 集成文档
- `:help vimscript` — Vimscript 文档
- `:help lsp` — LSP 文档
- `:help treesitter` — Tree-sitter 文档
- `src/nvim/lua/` — C-Lua 桥接层
- `runtime/lua/vim/` — Lua 标准库
