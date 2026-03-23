# libvim Buffer 算法分析与 Chez Scheme 重写评估

## 目录

1. [核心数据结构](#1-核心数据结构)
2. [B-tree 架构](#2-b-tree-架构)
3. [Data Block 内部布局](#3-data-block-内部布局)
4. [复杂度分析](#4-复杂度分析)
5. [辅助系统](#5-辅助系统)
6. [Chez Scheme 重写评估](#6-chez-scheme-重写评估)
7. [推荐数据结构方案](#7-推荐数据结构方案)

---

## 1. 核心数据结构

libvim（fork 自 Vim）的 buffer 存储使用 **B-tree（分支因子 ~128）**，而非数组或 rope。

关键特性：
- 树高通常 1-3 层（100 万行文件仅 2-3 层）
- 每个块 4096 字节（默认页大小）
- 所有指针操作 O(log N) 复杂度
- 内置 swap 文件持久化和崩溃恢复

源码位置（Neovim）：
- `src/nvim/memline.c` — 核心实现（~3000 行 C）
- `src/nvim/memline_defs.h` — 数据结构定义
- `src/nvim/memfile.c` — 虚拟内存/磁盘 I/O 层
- `src/nvim/memfile_defs.h` — memfile 结构定义
- `src/nvim/buffer_defs.h` — buf_T 结构（第 525 行嵌入 memline_T）

---

## 2. B-tree 架构

```
                    ┌─────────────────────┐
                    │   Pointer Block (根)  │
                    │  pe[0]: 500 lines    │
                    │  pe[1]: 500 lines    │
                    │  pe[2]: 500 lines    │
                    └───┬──────┬──────┬────┘
                        │      │      │
              ┌─────────┘      │      └─────────┐
              ▼                ▼                ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  Data Block   │ │  Data Block   │ │  Data Block   │
     │ [index array] │ │ [index array] │ │ [index array] │
     │ [free space]  │ │ [free space]  │ │ [free space]  │
     │ [text 反向存] │ │ [text 反向存] │ │ [text 反向存] │
     └──────────────┘ └──────────────┘ └──────────────┘
```

### 三级块结构

**Block 0（元数据块）**
- swap 文件头，用于崩溃恢复
- 存储：文件修改时间、inode、PID、用户名、主机名、文件名

**Pointer Block（内部节点）**
- 根节点在 block 1
- 包含指向子块的指针数组
- 每个 entry (`pointer_entry`) 包含：
  - `pe_bnum`：子块编号
  - `pe_line_count`：该子树的总行数
  - `pe_old_lnum`：用于恢复
  - `pe_page_count`：子块占用的页数
- 容量公式：`(page_size - sizeof(PTR_BL)) / sizeof(PTR_EN) + 1 ≈ 128`

**Data Block（叶节点）**
- 存储实际文本行
- 每块最多 ~800 行（MLCS_MAXL 常量）
- 容量取决于平均行长和页大小

---

## 3. Data Block 内部布局

关键设计：**文本反向存储**

```
┌─────────────────────────────┐ 低地址
│ Header                      │
│   db_id (标识符)             │
│   db_free (空闲字节数)       │
│   db_txt_start (文本起始)    │
│   db_txt_end (文本结束)      │
│   db_line_count (行数)       │
├─────────────────────────────┤
│ index[0] → line 1 offset    │
│ index[1] → line 2 offset    │  INDEX_SIZE = 4 字节/条目
│ ...                         │
│ index[n] → line N offset    │
├─────────────────────────────┤
│                             │
│       Free Space            │
│    (index 向下增长 ↓)        │
│    (text 向上增长 ↑)         │
│                             │
├─────────────────────────────┤
│ "line N text\0"             │
│ ...                         │
│ "line 2 text\0"             │
│ "line 1 text\0"             │ ← 最先的行在最高地址
└─────────────────────────────┘ 高地址
```

这种设计的好处：插入一行只需：
1. 移动 index 条目（小的、定长操作）
2. 在文本区追加文本
3. 无需按行号重排文本

---

## 4. 复杂度分析

| 操作 | 最佳 | 最坏 | 说明 |
|------|------|------|------|
| 读取行 `ml_get` | O(1) | O(log₁₂₈ N) | 缓存命中时 O(1) |
| 插入行 `ml_append` | O(1) | O(log N) 摊还 | 块满时触发分裂 |
| 删除行 `ml_delete` | O(1) | O(log N) | 需更新父节点行数 |
| 行号→字节偏移 | O(1) | O(chunks) | 通过 chunk 系统加速 |

### 关键函数

**`ml_get_buf(buf, lnum, will_change)`**（memline.c:1834）
1. 检查缓存 `ml_line_lnum` — O(1)
2. 调用 `ml_find_line()` 遍历树
3. 利用搜索栈缓存加速顺序访问

**`ml_find_line(buf, lnum, action)`**（memline.c:2922）
1. 检查当前锁定块范围（O(1) 命中）
2. 检查 `ml_stack` 缓存
3. 从根（block 1）开始，按 `pe_line_count` 逐层下降
4. 栈记录路径，加速后续访问

**`ml_append_int()`**（memline.c:1966）
1. 找到目标块
2. 有空间 → 插入 index + text
3. 无空间 → 分裂块，更新父指针块
4. 父节点满 → 递归向上分裂

---

## 5. 辅助系统

### 5.1 Memfile 层（虚拟内存）

管理物理文件 I/O 和内存缓存：
- 哈希表快速查找缓存块
- MRU 链表驱动缓存淘汰
- 负块号跟踪未保存变更
- 页对齐的文件访问

### 5.2 Memline_T 结构

```c
typedef struct memline {
  linenr_T ml_line_count;        // 总行数
  memfile_T *ml_mfp;             // 关联的 memfile

  // 搜索栈（树遍历路径缓存）
  infoptr_T *ml_stack;
  int ml_stack_top;
  int ml_stack_size;

  // 行缓存（单行热缓存）
  linenr_T ml_line_lnum;
  char_u *ml_line_ptr;

  // 块缓存（当前锁定块）
  bhdr_T *ml_locked;
  linenr_T ml_locked_low;
  linenr_T ml_locked_high;

  // Chunk 系统（行号↔字节偏移加速）
  chunksize_T *ml_chunksize;
  int ml_numchunks;
  int ml_usedchunks;
} memline_T;
```

### 5.3 Chunk 系统

- 每 ~800 行一个 chunk
- 每个 chunk 记录累计行数和字节数
- 避免树遍历即可完成 `byte2line()` / `line2byte()`

### 5.4 崩溃恢复

- Block 0 存储文件元数据
- swap 文件记录所有未保存变更
- `vim -r` 可从 swap 恢复

---

## 6. Chez Scheme 重写评估

### 完整复制 Vim memline：不推荐

| 方面 | 评估 |
|------|------|
| 代码量 | ~3000 行 C → ~2000+ 行 Scheme |
| 复杂度 | 深度耦合 swap 文件、磁盘 I/O、LRU 缓存 |
| 必要性 | 为 1990 年代内存稀缺设计，现代机器不需要 |
| FFI 开销 | 如直接绑定 C 库，大量指针操作在 Scheme 中繁琐 |

### 为什么不需要完整复制

Vim 的 memline 系统解决的核心问题是 **"文件大于可用内存"**。设计目标包括：
- 只将活跃块加载到内存
- LRU 淘汰冷块到 swap 文件
- 崩溃后从 swap 文件恢复

2026 年的场景下：
- 内存充足，整个文件可常驻内存
- 操作系统虚拟内存已处理大部分换页
- 编辑器自带自动保存，崩溃恢复需求降低

---

## 7. 推荐数据结构方案

### 方案对比

| 方案 | 适用场景 | Scheme 实现量 | 复杂度 |
|------|----------|--------------|--------|
| Vector of strings | Claude Code 风格 UI | ~100 行 | 插入/删除 O(N) |
| Gap Buffer | 轻量文本编辑器 | ~300 行 | 光标附近操作 O(1) |
| Piece Table | 通用编辑器（VS Code 用） | ~500 行 | 插入 O(1)，读取 O(pieces) |
| Rope | GB 级大文件 | ~1500 行 | 所有操作 O(log N) |

### 方案 1：Vector of Strings（最简单）

```scheme
(define-record-type buffer
  (fields (mutable lines)      ; vector of strings
          (mutable line-count)
          (mutable modified?)))

(define (buffer-get-line buf lnum)
  (vector-ref (buffer-lines buf) (- lnum 1)))

(define (buffer-insert-line! buf lnum text)
  (let* ((old (buffer-lines buf))
         (len (vector-length old))
         (new (make-vector (+ len 1))))
    (do ((i 0 (+ i 1)))
        ((= i (- lnum 1)))
      (vector-set! new i (vector-ref old i)))
    (vector-set! new (- lnum 1) text)
    (do ((i (- lnum 1) (+ i 1)))
        ((= i len))
      (vector-set! new (+ i 1) (vector-ref old i)))
    (buffer-lines-set! buf new)
    (buffer-line-count-set! buf (+ (buffer-line-count buf) 1))
    (buffer-modified?-set! buf #t)))
```

适合 <100K 行，对 Claude Code 风格应用绰绰有余。

### 方案 2：Gap Buffer

```scheme
(define-record-type gap-buffer
  (fields (mutable lines)       ; vector，预分配大于实际行数
          (mutable gap-start)   ; gap 起始位置
          (mutable gap-end)     ; gap 结束位置
          (mutable line-count)))

;; 光标附近插入/删除 O(1)，随机位置 O(N) 移动 gap
```

### 方案 3：Piece Table

```scheme
(define-record-type piece
  (fields source     ; 'original 或 'add
          start      ; 在对应 buffer 中的偏移
          length))   ; 字符数

(define-record-type piece-table
  (fields original    ; 原始文本（不可变 string）
          (mutable add-buffer)  ; 追加 buffer
          (mutable pieces)))    ; piece 的有序序列
```

### 实际建议

对于大多数场景，从 **vector of strings** 开始，按需升级：

```
vector of strings → gap buffer → piece table → rope
       ↑
  从这里开始，够用就不升级
```
