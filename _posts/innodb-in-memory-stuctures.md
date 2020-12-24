---
layout: post
title: "Innodb-tmp"
author: "Dapaopao"
tags: ["innodb"]
---

# InnoDB In-Memory Structures

## Buffer Pool

相关代码文件：`include/buf0buf.h`，

```cpp
/** The buffer control block structure */
struct buf_block_t {
  /** page information; this must be the first field, so
  that buf_pool->page_hash can point to buf_page_t or buf_block_t */
  buf_page_t page;

  /** pointer to buffer frame which is of size UNIV_PAGE_SIZE, and aligned
  to an address divisible by UNIV_PAGE_SIZE */
  byte *frame;
  
  ...
};
```



## 内存管理

buffer allocation,16kb

dynamic allocation

```cpp
struct ut_new_pfx_t {
  
};

class ut_allocator {
  
};
```



#include/univ.i：Version control for database, common definitions, and include files

#include/ut0new.h

#include/mem0mem.h：allocate

#include/mem0mem.ic：mem_heap_alloc

#include/ut0dbg.h

#include/ut0lst.h：struct ut_list_base



mem/memory.cc：mem_heap_add_block



buf/buf0buf.cc：buf_block_alloc

buf/buf0lru.cc：buf_LRU_get_free_block



include/log0recv.ic：recv_recovery_is_on()，是否处于recovery状态(apply redo log records)。







## buddy

相关代码文件：`include/buf0buf.h`，`include/buf0buddy.ic`，`buf/buf0buddy.cc`。

innodb支持创建压缩页以减少数据占用的磁盘空间，支持1K, 2K, 4K 、8K和16k大小的page。buddy allocator用于管理压缩page的内存空间，提高内存使用率和性能。

`buddy`内存管理算法，linux内核用于解决外部碎片的一种手段。innodb的实现在算法上与buddy的基本实现并无什么区别，所支持最小的内存块为1k，最大为16k，每种内存块维护一个链表，多种内存块链表组成了zip_free链表。

https://en.wikipedia.org/wiki/Buddy_memory_allocation

分配入口在**buf_buddy_alloc_low**，先尝试从zip_free[i]中获取所需大小的内存块，如果当前链表中没有，则尝试从更大的内存块链表中获取，获取成功则进行切分，一部分返回另一块放入对应free链表中，实际上是buf_buddy_alloc_zip的一个递归调用，只是传入的i不断增加。如果一直递归到16k的块都没法满足，则从buffer pool中新申请一块大内存块，并将其按照伙伴关系进行(比如现分配了16，需要2k，先切分8k,8k，再将其中一个8k切分为4k,4k，再将其中4k切分为2k,2k)切分直到满足分配请求。

释放入口在**buf_buddy_free_low**，为了避免碎片在释放的时候多做了一些事情。在释放一个内存块的时候没有直接放回对应链表中，而是先查看其伙伴是不是free的，如果是则进行合并，再尝试对合并后的内存块进行合并。如果其伙伴是在USED的状态，这里做了一次relocate操作，将其内容拷贝到其它free的block块上，再进行对它合并。这种做法有效减少了碎片的存在，但拷贝这种操作也降低了性能。

```cpp
/** Struct that is embedded in the free zip blocks */
struct buf_buddy_free_t {
  union {
    ulint size; /*!< size of the block */
    byte bytes[FIL_PAGE_DATA];
    /*!< stamp[FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID]
    == BUF_BUDDY_FREE_STAMP denotes a free
    block. If the space_id field of buddy
    block != BUF_BUDDY_FREE_STAMP, the block
    is not in any zip_free list. If the
    space_id is BUF_BUDDY_FREE_STAMP then
    stamp[0] will contain the
    buddy block size. */
  } stamp;

  buf_page_t bpage; /*!< Embedded bpage descriptor */
  UT_LIST_NODE_T(buf_buddy_free_t) list;
  /*!< Node of zip_free list */
};

// Binary buddy allocator for compressed pages

/** Allocate a block.
@param[in,out]	buf_pool	buffer pool instance
@param[in]	i		index of buf_pool->zip_free[]
                                or BUF_BUDDY_SIZES
@return allocated block, never NULL */
void *buf_buddy_alloc_low(buf_pool_t *buf_pool, ulint i)

/** Deallocate a block.
@param[in]	buf_pool	buffer pool instance
@param[in]	buf		block to be freed, must not be pointed to
                                by the buffer pool
@param[in]	i		index of buf_pool->zip_free[],
                                or BUF_BUDDY_SIZES
@param[in]	has_zip_free	whether has zip_free_mutex */
// 释放申请的buddy block回buffer pool中的zip_free。
void buf_buddy_free_low(buf_pool_t *buf_pool, void *buf, ulint i,
                        bool has_zip_free);

/** Deallocate a buffer frame of UNIV_PAGE_SIZE.
@param[in]	buf_pool	buffer pool instance
@param[in]	buf		buffer frame to deallocate */
// 从zip_hash中取出page，放回buffer pool的free list。
static void buf_buddy_block_free(buf_pool_t *buf_pool, void *buf);
```



## Utilities

#### 1. Hash Table

相关代码文件：`include/hash0hash.h`，`include/hash0hash.ic`，`ha/hash0hash.cc`。

哈希表工具类，用`marco`实现了查找，更新等函数。

几个注意的点：

1. 拉链法解决哈希冲突。
2. 提供了分段`mutex`，`rw_lock`等选项提供线程安全。
3. 新增的值通过`mem_heap_t`管理内存，提供`HASH_DELETE_AND_COMPACT`函数，当删除一个值后，将`mem_heap_t`中顶部的值填充到被删除的地方。

```cpp
struct hash_cell_t {
	void *node; /*!< hash chain node, NULL if none */
};

struct hash_table_t {
  enum hash_table_sync_t type; /*!< type of hash_table. */
  ulint n_cells;      /* number of cells in the hash table */
  hash_cell_t *cells; /*!< pointer to cell array */

  ulint n_sync_obj; /* if sync_objs != NULL, then
                 the number of either the number
                 of mutexes or the number of
                 rw_locks depending on the type.
                 Must be a power of 2 */
  union {
    ib_mutex_t *mutexes; /* NULL, or an array of mutexes
                         used to protect segments of the
                         hash table */
    rw_lock_t *rw_locks; /* NULL, or an array of rw_lcoks
                        used to protect segments of the
                        hash table */
  } sync_obj;

  mem_heap_t **heaps; /*!< if this is non-NULL, hash
                      chain nodes for external chaining
                      can be allocated from these memory
                      heaps; there are then n_mutexes
                      many of these heaps */
  mem_heap_t *heap;
};
```



#### 2. Linked List

相关代码文件：`include/ut0lst.h`。

一个`marco`实现的双向链表工具。

```cpp
template <typename Type>
struct ut_list_node {
  Type *prev; /*!< pointer to the previous
              node, NULL if start of list */
  Type *next; /*!< pointer to next node,
              NULL if end of list */
  void reverse() {
    Type *tmp = prev;
    prev = next;
    next = tmp;
  }
};

template <typename Type, typename NodePtr>
struct ut_list_base {
  typedef Type elem_type;
  typedef NodePtr node_ptr;
  
  ulint count{0};            /*!< count of nodes in list */
  elem_type *start{nullptr}; /*!< pointer to list start,
                             NULL if empty */
  elem_type *end{nullptr};   /*!< pointer to list end,
                             NULL if empty */
  node_ptr node{nullptr};    /*!< Pointer to member field
                             that is used as a link node */
  void reverse() {
    Type *tmp = start;
    start = end;
    end = tmp;
  }
};
```



#### 3. Codecs

相关代码文件：`include/mach0data.ic`。

编解码工具，使用`big endian`对文件的数字和内存的数字互相转换。











## 参考资料

https://zhuanlan.zhihu.com/p/111718288



