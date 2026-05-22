# Linux page cache 笔记

## 一、page cache 是什么"对象"

page cache 不是一个单一对象，而是**每个 inode 一份**的"文件偏移 → 物理页帧"的索引表。核心数据结构：

```
struct inode
   └── i_mapping  ──►  struct address_space
                         ├── host           (回指 inode)
                         ├── a_ops          (readpage/writepage/...)
                         ├── i_pages        ◄── XArray:  pgoff_t → struct page *
                         └── nrpages, flags, ...
```

被缓存的每一页就是一个**普通的物理页帧**（`struct page`，对应一个 4 KiB RAM 帧），上面挂着：
- `page->mapping` 回指 address_space
- `page->index` 在文件中的页偏移
- 标志位：`PG_uptodate`（内容与磁盘一致）、`PG_dirty`（待回写）、`PG_locked` 等
- 同时挂在 LRU 链表上供 reclaim 扫描

所以"page cache"= **address_space 的 XArray + 一堆带标记的普通 RAM 页帧**。它不在某个固定地址，而是散落在物理内存里，由 XArray 按 `(inode, file offset)` 查找到。

## 二、它在内核中处在哪一层

```
        用户进程
   read/write/mmap (syscall)
           │
           ▼
   ┌──────────────────┐
   │      VFS         │   file->f_op->read_iter / write_iter / mmap
   └────────┬─────────┘
            ▼
   ┌──────────────────┐
   │   filemap.c      │   ←—— page cache 的入口层
   │  (address_space) │       filemap_read / filemap_fault
   └────────┬─────────┘       generic_perform_write
            │  miss
            ▼
   ┌──────────────────┐
   │  a_ops->readpage │   文件系统（ext4/xfs/...）实现
   │  ->writepage     │
   └────────┬─────────┘
            ▼
   ┌──────────────────┐
   │   block layer    │   bio → request → driver → 磁盘
   └──────────────────┘
```

page cache 夹在 **VFS 与文件系统/块层之间**，是"按页缓存文件数据"的薄层。

## 三、三条数据通路的具体流程

### 1) `read(fd, buf, n)`
```
vfs_read
 └─ generic_file_read_iter
     └─ filemap_read
         ├─ filemap_get_pages
         │   ├─ xa_load(&mapping->i_pages, index)   ← 查 XArray
         │   └─ miss: page_cache_ra_unbounded
         │           └─ a_ops->readpage → submit_bio → 等 PG_uptodate
         └─ copy_page_to_iter(page, ..., iter)      ← memcpy
             └─ 源地址 = page_address(page)  即 direct map
```
关键点：**copy 的源是 page 的内核 direct-map 虚拟地址**，目的是用户 buf。用户态从未"看见"这个物理页。

### 2) `write(fd, buf, n)`（带缓冲）
```
generic_perform_write
 ├─ grab_cache_page_write_begin   (查不到就分配并插入 XArray)
 ├─ a_ops->write_begin            (可能预读旧内容)
 ├─ copy_from_user(page_address(page)+off, buf, k)
 └─ a_ops->write_end              (SetPageDirty, 标 inode dirty)
```
**只标脏，不落盘**。之后由 `wb_workfn` → `writepages` → `a_ops->writepage` 异步回写。

### 3) `mmap` + 缺页
```
mmap()                          ← 仅建立 vma，不映射页
  └─ file->f_op->mmap = generic_file_mmap
      └─ vma->vm_ops = &generic_file_vm_ops

第一次访问 → page fault:
  do_user_addr_fault → handle_mm_fault → do_fault
    └─ __do_fault → vma->vm_ops->fault = filemap_fault
        ├─ 在 i_pages 中找页；miss 则 readpage
        └─ vmf->page = 该 cache 页
    └─ finish_fault → 把该页 PFN 写入用户 PTE
```
此后用户态 load/store 直接打到该物理页，与内核的 direct map 指向**同一个 RAM 帧**。

## 四、那它的作用到底是什么（不和权限挂钩）

它的价值在**性能与一致性**，与权限正交：

1. **避免重复 I/O**：同一文件被多次 read、被多个进程打开/mmap，只读一次磁盘。
2. **统一视图**：`read/write` 与 `MAP_SHARED` 看的是同一份页帧，写穿透 → 立即可见，无需 flush。
3. **写聚合 / 延迟回写**：把许多小写合并成少量大 bio，磁盘吞吐显著提升。
4. **预读**：基于访问模式提前把后续页读入。
5. **作为内存与磁盘之间的统一货架**：reclaim 时按 LRU 把干净页直接丢弃、脏页回写后丢弃，让"文件数据"与"匿名内存"在内存压力下统一调度。

权限检查发生在**进入这一层之前**（open 时的 inode_permission、syscall 入口的 f_mode、mmap 时的 vm_flags 协商）和**离开这一层之后**（用户访问时 MMU 查 PTE）。page cache 处在中间，专心做"数据缓冲与索引"。
