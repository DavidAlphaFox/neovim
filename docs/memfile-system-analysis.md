# Memfile 系统：Vim 的用户态虚拟内存

## 目录

1. [核心思想](#1-核心思想)
2. [数据结构](#2-数据结构)
3. [机制一：负数/正数块号 — 延迟写入](#3-机制一负数正数块号--延迟写入)
4. [机制二：Lock/Unlock 协议](#4-机制二lockunlock-协议)
5. [机制三：MRU Used List — 内存淘汰](#5-机制三mru-used-list--内存淘汰)
6. [机制四：Hash Table — O(1) 块查找](#6-机制四hash-table--o1-块查找)
7. [机制五：Free List — 磁盘空间回收](#7-机制五free-list--磁盘空间回收)
8. [机制六：写入时填补空洞](#8-机制六写入时填补空洞)
9. [整体数据流](#9-整体数据流)
10. [设计总结](#10-设计总结)

---

## 1. 核心思想

Memfile 本质上是一个 **用户态的虚拟内存管理器**，它让上层（memline/B-tree）可以像操作内存一样操作块，而不需要关心这些块实际上是在内存还是磁盘上。

```
上层调用者 (memline)
    │
    │  "给我 block #37"  /  "新建一个 block"
    ▼
┌─────────────────────────────────────────────┐
│              Memfile 系统                     │
│                                              │
│  ┌──────────┐    ┌───────────┐    ┌───────┐ │
│  │ Hash Table│    │ Used List │    │ Free  │ │
│  │ (快速查找) │    │ (MRU 淘汰)│    │ List  │ │
│  └──────────┘    └───────────┘    └───────┘ │
│                       │                      │
│              ┌────────┴────────┐             │
│              ▼                 ▼             │
│         [内存中]          [磁盘上]           │
│         locked/          swap file           │
│         cached blocks    (.swp)              │
└─────────────────────────────────────────────┘
```

源码位置（Neovim）：
- `src/nvim/memfile.c` — 实现（~950 行 C）
- `src/nvim/memfile_defs.h` — 数据结构定义

公开 API：

| 函数 | 作用 |
|------|------|
| `mf_open()` | 打开新的或已有的 memfile |
| `mf_open_file()` | 为已有 memfile 打开 swap 文件 |
| `mf_close()` | 关闭（并删除）memfile |
| `mf_new()` | 创建新块并锁定 |
| `mf_get()` | 获取已有块并锁定 |
| `mf_put()` | 解锁块，可标记为脏 |
| `mf_free()` | 释放块 |
| `mf_sync()` | 将脏块同步到磁盘 |
| `mf_release_all()` | 释放尽可能多的内存 |
| `mf_trans_del()` | 查询负→正块号翻译 |

---

## 2. 数据结构

### memfile_T — 内存文件

```c
typedef struct memfile {
  char_u *mf_fname;              // swap 文件名
  char_u *mf_ffname;             // swap 文件全路径
  int mf_fd;                     // 文件描述符（-1 表示纯内存模式）

  bhdr_T *mf_free_first;         // free list 头
  bhdr_T *mf_used_first;         // used list 头（MRU 端）
  bhdr_T *mf_used_last;          // used list 尾（LRU 端）

  mf_hashtab_T mf_hash;          // 块号 → bhdr_T 的哈希表
  mf_hashtab_T mf_trans;         // 负块号 → 正块号的翻译表

  blocknr_T mf_blocknr_max;      // 最大正块号 + 1
  blocknr_T mf_blocknr_min;      // 最小负块号 - 1
  blocknr_T mf_neg_count;        // 负块号数量
  blocknr_T mf_infile_count;     // 文件中已有的页数

  unsigned mf_page_size;          // 页大小（默认 4096）
  bool mf_dirty;                  // 是否有脏块
} memfile_T;
```

### bhdr_T — 块头

```c
typedef struct bhdr {
  mf_hashitem_T bh_hashitem;     // 侵入式 hash 节点（包含 bh_bnum）
  struct bhdr *bh_next;          // used/free list 链表指针
  struct bhdr *bh_prev;          // used list 反向指针
  void *bh_data;                 // 指向实际数据内存
  unsigned bh_page_count;        // 占用页数
  unsigned bh_flags;             // BH_DIRTY | BH_LOCKED
} bhdr_T;
```

### mf_hashtab_T — 侵入式哈希表

```c
typedef struct mf_hashtab {
  size_t mht_mask;               // 桶数 - 1（用于取模）
  size_t mht_count;              // 元素数量
  mf_hashitem_T **mht_buckets;   // 桶数组
  mf_hashitem_T *mht_small_buckets[64]; // 初始内嵌桶（避免小表分配）
} mf_hashtab_T;
```

### mf_blocknr_trans_item_T — 块号翻译条目

```c
typedef struct mf_blocknr_trans_item {
  mf_hashitem_T nt_hashitem;     // key = nt_old_bnum（负数）
  blocknr_T nt_new_bnum;         // 翻译后的正数块号
} mf_blocknr_trans_item_T;
```

---

## 3. 机制一：负数/正数块号 — 延迟写入

这是 memfile 最精妙的设计。

### 块号空间

```
块号空间:
   ... -3  -2  -1  │  0  1  2  3  4 ...
   ◄── 仅内存的块 ──┤── 已分配磁盘位置的块 ──►
       (负数)       │       (正数)
```

**规则**：
- **正数块号** = 已经有磁盘位置，块号 == swap 文件中的页号
- **负数块号** = 只存在于内存，还没写过磁盘

### 为什么这样设计？

新建块时（`mf_new`），如果传 `negative=true`，获得一个负数块号（从 -1 递减）。这个块只存在于内存中。只有在需要写入磁盘时（内存不够或 sync），才通过 `mf_trans_add()` 分配一个正数块号。

```c
// mf_new() 中的关键逻辑：
if (negative) {
    hp->bh_bnum = mfp->mf_blocknr_min--;   // -1, -2, -3 ...
    mfp->mf_neg_count++;
} else {
    hp->bh_bnum = mfp->mf_blocknr_max;     // 追加到文件末尾
    mfp->mf_blocknr_max += page_count;
}
```

### 翻译表

B-tree 的 pointer block 中存的是负数块号引用，但写入磁盘时块号变正了。**翻译表** (`mf_trans`) 解决这个问题：

```
翻译表 (mf_trans):
  old_bnum (-3) ──→ new_bnum (7)
  old_bnum (-1) ──→ new_bnum (5)

上层仍然用 -3 引用这个块
    │
    ▼ mf_trans_del(-3)
    返回 7，然后用 7 去 hash 表找真正的块
```

```c
// mf_trans_add: 负→正转换
static int mf_trans_add(memfile_T *mfp, bhdr_T *hp) {
    // 分配新的正数块号
    new_bnum = mfp->mf_blocknr_max;
    mfp->mf_blocknr_max += page_count;

    // 记录翻译: old_bnum(-3) → new_bnum(7)
    np->nt_old_bnum = hp->bh_bnum;
    np->nt_new_bnum = new_bnum;

    // 更新 hash 表中的 key
    mf_rem_hash(mfp, hp);
    hp->bh_bnum = new_bnum;
    mf_ins_hash(mfp, hp);
}
```

---

## 4. 机制二：Lock/Unlock 协议

所有块操作的核心流程：

```
调用者想读 block #N
    │
    ▼
mf_get(mfp, N, page_count)
    │
    ├── 在 hash 表中找到？
    │   ├── YES → 从 used list 移到头部（MRU）
    │   │         设置 BH_LOCKED
    │   │         返回 bhdr_T*
    │   │
    │   └── NO  → 块号有效且在文件中？
    │       ├── YES → 分配内存，从磁盘读入 (mf_read)
    │       │         插入 hash + used list
    │       │         设置 BH_LOCKED
    │       │         返回 bhdr_T*
    │       │
    │       └── NO  → 返回 NULL
    │
调用者操作完毕
    │
    ▼
mf_put(mfp, hp, dirty=true, infile=false)
    │
    ├── 清除 BH_LOCKED
    ├── 如果 dirty → 设置 BH_DIRTY
    └── 如果 infile → 调用 mf_trans_add（负→正转换）
```

**为什么需要 Lock？** 被锁定的块不会被淘汰到磁盘。上层正在操作这个块的内存，如果被淘汰，指针就悬空了。

---

## 5. 机制三：MRU Used List — 内存淘汰

```
mf_used_first (最近使用)                    mf_used_last (最久未用)
     │                                              │
     ▼                                              ▼
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ blk 5│◄──►│ blk 2│◄──►│ blk 9│◄──►│ blk 1│◄──►│ blk 4│
  │LOCKED│    │      │    │DIRTY │    │      │    │      │
  └──────┘    └──────┘    └──────┘    └──────┘    └──────┘
                                                     ↑
                                              最先被淘汰的候选
```

### 淘汰策略

内存不足时 (`mf_release_all`)，从**尾部**（LRU 端）开始淘汰：

```c
for (hp = mfp->mf_used_last; hp != NULL; ) {
    if (!(hp->bh_flags & BH_LOCKED)           // 没被锁定
        && (!(hp->bh_flags & BH_DIRTY)        // 不脏（不需写）
            || mf_write(mfp, hp) != FAIL)) {  // 或者成功写出
        // 释放内存
        mf_rem_used(mfp, hp);
        mf_rem_hash(mfp, hp);
        mf_free_bhdr(hp);
        hp = mfp->mf_used_last;    // 重新从尾部开始
    } else {
        hp = hp->bh_prev;          // 跳过锁定的块
    }
}
```

### MRU Promotion

每次 `mf_get` 命中缓存时，把块移到链表头部：

```c
// mf_get 中：
mf_rem_used(mfp, hp);   // 从当前位置摘除
mf_ins_used(mfp, hp);   // 插入头部（最近使用）
```

---

## 6. 机制四：Hash Table — O(1) 块查找

```
hash(block_nr) = block_nr & mask

buckets[0]:  blk 0  → blk 64  → blk 128 → NULL
buckets[1]:  blk 1  → NULL
buckets[2]:  blk 2  → blk 66  → NULL
...
buckets[63]: blk 63 → blk -1  → NULL
```

### 特点

- **侵入式** (intrusive)：`bhdr_T` 的第一个字段就是 `mf_hashitem_T`，零额外分配
- 负数块号也能 hash（`& mask` 对负数同样有效）
- 自动扩容：当 `count >> 6 > mask` 时翻倍（平均桶长超过 64 时扩容）
- 初始 64 个桶内嵌在结构体中 (`mht_small_buckets`)，小表零堆分配

### 扩容 rehash

```c
static void mf_hash_grow(mf_hashtab_T *mht) {
    // 新桶数 = 旧桶数 × 2
    // 遍历旧桶，按高位 bit 分流到两个新桶
    // 保持桶内顺序（MRU 在前）
    for (size_t i = 0; i <= mht->mht_mask; i++) {
        // 每个旧桶的元素按 key 的高位 bit 分到 2 个新桶
        size_t j = (mhi->mhi_key >> shift) & 1;
        buckets[i + (j << shift)] = mhi;
    }
}
```

---

## 7. 机制五：Free List — 磁盘空间回收

```
当一个正数块被释放（如 B-tree 合并时）：
    │
    ▼
mf_free(mfp, hp)
    │
    ├── 负数块 → 直接 free 内存（磁盘上没有对应空间）
    │             mf_neg_count--
    │
    └── 正数块 → 放入 free list（记住这个磁盘位置可复用）
         │
         日后 mf_new() 或 mf_trans_add() 时：
         "需要一个正数块号？free list 里有空位，直接复用"
```

Free list 还支持部分复用：

```c
// mf_new() 中：
if (freep->bh_page_count > page_count) {
    // free 块有 5 页，只需要 1 页
    hp->bh_bnum = freep->bh_bnum;       // 取前 1 页
    freep->bh_bnum += page_count;        // free 块起始后移
    freep->bh_page_count -= page_count;  // free 块缩小
}
```

这避免了 swap 文件无限增长——被删除的 B-tree 节点释放的磁盘空间可以被新块复用。

---

## 8. 机制六：写入时填补空洞

`mf_write` 中有一段精巧的逻辑，确保 swap 文件无空洞：

```c
// 如果要写 block 7，但文件只有 5 个块（0-4），
// 中间的 block 5、6 是"空洞"
for (;;) {
    nr = hp->bh_bnum;
    if (nr > mfp->mf_infile_count) {
        // 先写 mf_infile_count 位置的块来填补空洞
        nr = mfp->mf_infile_count;
        hp2 = mf_find_hash(mfp, nr);  // 这个位置的块可能存在
    }
    // 写 hp2（如果存在）或用 dummy 数据填充
    // 更新 mf_infile_count
    // 直到 nr == hp->bh_bnum（目标块写入完成）
}
```

保证 swap 文件没有空洞，崩溃恢复时可以顺序读取。

---

## 9. 整体数据流

### 一次编辑操作的完整路径

```
编辑操作: 用户输入 "dd" 删除一行
    │
    ▼
memline 层: ml_delete()
    │
    ├── mf_get(block_nr)    ← 获取 data block，锁定
    │   [memfile: hash查找 → 缓存命中/磁盘读取 → 返回锁定块]
    │
    ├── 修改 data block 内容（删除行、调整 index）
    │
    ├── mf_put(dirty=true)  ← 解锁，标记脏
    │   [memfile: 清LOCKED、设DIRTY]
    │
    ├── mf_get(parent_ptr_block) ← 更新父 pointer block 的行数
    ├── 修改 pe_line_count
    ├── mf_put(dirty=true)
    │
    ▼
后台/定时: mf_sync()
    │
    ├── 遍历 used list（从尾到头）
    ├── 找 DIRTY 块 → mf_write → 写入 swap 文件
    ├── 清除 DIRTY 标记
    └── 如果是负数块 → mf_trans_add 转为正数 → 写入
```

### mf_sync 的策略

```c
int mf_sync(memfile_T *mfp, int flags) {
    // 从尾到头遍历（降低不一致概率）
    for (hp = mfp->mf_used_last; hp != NULL; hp = hp->bh_prev) {
        if (hp is dirty && should_sync) {
            mf_write(mfp, hp);     // 写入磁盘
        }
        if (flags & MFS_STOP) {
            if (os_char_avail())   // 有用户输入？停止 sync
                break;             // 优先响应用户
        }
    }
    if (flags & MFS_FLUSH) {
        os_fsync(mfp->mf_fd);     // fsync 确保落盘
    }
}
```

支持的 flags：
- `MFS_ALL` — 同步所有脏块（包括负数块号）
- `MFS_STOP` — 检测到用户输入时停止（保持响应性）
- `MFS_FLUSH` — fsync 确保数据落盘（防崩溃丢失）
- `MFS_ZERO` — 只写 block 0（恢复元数据）

---

## 10. 设计总结

| 机制 | 解决的问题 |
|------|-----------|
| 负/正块号 | 延迟写入——大量临时修改不必立刻持久化 |
| 翻译表 | 块号变化时上层无感知，引用仍然有效 |
| Lock/Unlock | 保护正在操作的内存不被淘汰 |
| MRU Used List | 低成本的缓存淘汰策略 |
| Hash Table | O(1) 块查找，侵入式零额外分配 |
| Free List | 磁盘空间回收复用，支持部分复用 |
| 填补空洞 | swap 文件连续，可靠恢复 |
| MFS_STOP | sync 期间检测用户输入，优先保持响应性 |

### 与现代系统的对比

整个系统约 950 行 C，本质上是一个**带 LRU 缓存的块设备抽象层**，加上延迟分配 + 翻译的巧妙机制。

类比现代系统：
- 负/正块号 ≈ Linux 的 **延迟分配** (delayed allocation)
- 翻译表 ≈ 虚拟地址 → 物理地址的 **页表**
- Used List ≈ Linux 的 **LRU 页回收**
- Lock ≈ **页固定** (page pinning)
- Free List ≈ **空闲块管理**

它是 1990 年代"内存比文件小"背景下的精致工程，用不到 1000 行代码实现了一个完整的用户态虚拟内存系统。
