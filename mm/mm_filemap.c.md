# mm/filemap.c - Linux File-Backed Memory Mapping and Page Cache

## Overview

This file implements the generic file mapping semantics and page cache management for Linux, originally developed by Linus Torvalds in 1994-1999. It provides the critical interface between the virtual memory system and file systems, enabling memory-mapped files, page cache operations, and efficient file I/O through shared memory. The implementation supports most "normal" filesystems and handles the complex interactions between file data, memory pages, and virtual memory mappings.

## Historical Development

### Key Contributors and Milestones
- **Linus Torvalds (1994-1999)**: Original file mapping implementation
- **Bruno (1995)**: Shared mappings working implementation (August 15, 1995)
- **Ingo Molnar - Red Hat (1999)**: Page and buffer cache unification, SMP threading
- **Andrea Arcangeli - SUSE (1999)**: SMP-threaded pagemap-LRU implementation

### Evolution Timeline
- **November 30, 1994**: Shared mappings initial implementation
- **August 15, 1995**: Fully working shared mappings
- **May 21, 1999**: Page and buffer cache unification with SMP threading
- **1999**: SMP-threaded pagemap-LRU system
- **2000s**: Advanced readahead algorithms and NUMA optimizations
- **2010s**: Transparent huge page support and modern scalability improvements

### Design Philosophy
The file mapping system is built around the principle of unified page cache management, where file data and memory pages are seamlessly integrated to provide efficient file access through both traditional I/O and memory mapping interfaces.

## Core Concepts

### File Mapping Architecture

#### Memory Mapping Pipeline
```
File Operations → Address Space → Page Cache → Virtual Memory
       ↓              ↓             ↓             ↓
[File System]   [Inode Mapping]  [Physical Pages] [User Space]
```

#### Page Cache Hierarchy
```
Address Space (struct address_space)
  ↓
Page Tree (XArray-based radix tree)
  ↓
File Pages (struct folio/page)
  ↓
Physical Memory
```

#### Memory Mapping Types
- **Private Mappings**: Copy-on-write file mappings (MAP_PRIVATE)
- **Shared Mappings**: Shared file mappings (MAP_SHARED)
- **Anonymous Mappings**: Memory-only mappings without file backing
- **Special Mappings**: Device files and special file types

## Key Data Structures

### `struct address_space` - File Mapping Context
```c
struct address_space {
    struct inode           *host;           /* Owner inode */
    struct xarray          i_pages;         /* Page tree */
    struct rw_semaphore    invalidate_lock; /* Invalidation protection */
    gfp_t                  gfp_mask;        /* Allocation mask */
    
    atomic_t               i_mmap_writable; /* Writable mappings count */
    struct rb_root_cached  i_mmap;          /* Private mappings tree */
    struct rw_semaphore    i_mmap_rwsem;    /* Mapping protection */
    
    unsigned long          nrpages;         /* Total pages */
    unsigned long          nrexceptional;   /* Exceptional entries */
    pgoff_t                writeback_index; /* Writeback cursor */
    
    const struct address_space_operations *a_ops; /* Operations */
    unsigned long          flags;           /* Mapping flags */
    errseq_t              wb_err;          /* Writeback error */
    
    spinlock_t            private_lock;     /* Private data protection */
    struct list_head      private_list;     /* Private data list */
    void                  *private_data;    /* Filesystem private data */
};
```

### Page Cache State Management
```c
/* Page states in cache */
#define PG_locked        0   /* Page is locked */
#define PG_referenced    1   /* Page was referenced */
#define PG_uptodate      2   /* Page data is valid */
#define PG_dirty         3   /* Page needs writeback */
#define PG_lru           4   /* Page is on LRU list */
#define PG_active        5   /* Page is on active LRU */
#define PG_workingset    7   /* Page was in working set */
#define PG_waiters       8   /* Page has waiters */
#define PG_writeback     15  /* Page is under writeback */
#define PG_mappedtodisk  16  /* Page blocks allocated on disk */
#define PG_reclaim       17  /* Page marked for reclaim */
#define PG_readahead     18  /* Page marked for readahead */
```

### File Mapping Operations
```c
struct address_space_operations {
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    int (*read_folio)(struct file *, struct folio *);
    int (*writepages)(struct address_space *, struct writeback_control *);
    
    bool (*dirty_folio)(struct address_space *, struct folio *);
    void (*readahead)(struct readahead_control *);
    
    int (*write_begin)(struct file *, struct address_space *mapping,
                      loff_t pos, unsigned len,
                      struct page **pagep, void **fsdata);
    int (*write_end)(struct file *, struct address_space *mapping,
                    loff_t pos, unsigned len, unsigned copied,
                    struct page *page, void *fsdata);
    
    sector_t (*bmap)(struct address_space *, sector_t);
    void (*invalidate_folio)(struct folio *, size_t offset, size_t len);
    bool (*release_folio)(struct folio *, gfp_t);
    void (*free_folio)(struct folio *);
    
    ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *);
    int (*migratepage)(struct address_space *, struct page *, struct page *,
                      enum migrate_mode);
    bool (*isolate_page)(struct page *, isolate_mode_t);
    void (*putback_page)(struct page *);
    int (*launder_folio)(struct folio *);
    bool (*is_partially_uptodate)(struct folio *, size_t from, size_t count);
    void (*is_dirty_writeback)(struct folio *, bool *dirty, bool *writeback);
    int (*error_remove_page)(struct address_space *, struct page *);
    int (*swap_activate)(struct swap_info_struct *sis, struct file *file,
                        sector_t *span);
    void (*swap_deactivate)(struct file *file);
    int (*swap_rw)(struct kiocb *iocb, struct iov_iter *iter);
};
```

## Core Functions

### Page Cache Management

#### `__filemap_add_folio()` - Add Folio to Page Cache
```c
noinline int __filemap_add_folio(struct address_space *mapping,
                                struct folio *folio, pgoff_t index, 
                                gfp_t gfp, void **shadowp)
```

**Purpose**: Core function to add a folio (page or compound page) to the page cache

**Addition Process**:
1. **Validation**: Verify folio state and mapping constraints
2. **Conflict Resolution**: Handle existing entries and shadow pages
3. **XArray Insertion**: Insert folio into the radix tree structure
4. **Statistics Update**: Update page cache and LRU statistics
5. **Reference Management**: Establish proper reference counting

**Key Features**:
```c
/* Handle shadow entries for workingset detection */
if (old) {
    if (order > 0 && order > forder) {
        /* Split large shadow entries */
        xas_try_split(&xas, old, order);
    }
    if (shadowp)
        *shadowp = old;
}

/* Store folio and update statistics */
xas_store(&xas, folio);
mapping->nrpages += nr;
if (!huge) {
    __lruvec_stat_mod_folio(folio, NR_FILE_PAGES, nr);
    if (folio_test_pmd_mappable(folio))
        __lruvec_stat_mod_folio(folio, NR_FILE_THPS, nr);
}
```

#### `filemap_add_folio()` - Public Interface for Page Cache Addition
```c
int filemap_add_folio(struct address_space *mapping, struct folio *folio,
                     pgoff_t index, gfp_t gfp)
```

**Public Interface Features**:
- **Memory Accounting**: Proper memory cgroup accounting
- **Error Handling**: Comprehensive error handling and cleanup
- **Shadow Handling**: Workingset shadow page management
- **LRU Integration**: Automatic LRU list addition

#### `page_cache_delete()` - Remove Page from Cache
```c
static void page_cache_delete(struct address_space *mapping,
                             struct folio *folio, void *shadow)
```

**Deletion Process**:
1. **XArray Removal**: Remove folio from radix tree
2. **Shadow Installation**: Install shadow entry for workingset tracking
3. **Statistics Update**: Update page count statistics
4. **Reference Cleanup**: Clear mapping association

### Page Fault Handling

#### `filemap_fault()` - File-Backed Page Fault Handler
```c
vm_fault_t filemap_fault(struct vm_fault *vmf)
```

**Purpose**: Handle page faults for memory-mapped files

**Fault Resolution Process**:
1. **Bounds Checking**: Verify access is within file bounds
2. **Cache Lookup**: Search page cache for existing page
3. **Readahead Decision**: Determine if readahead is beneficial
4. **Page Allocation**: Allocate page if not in cache
5. **I/O Initiation**: Start I/O if page data not available
6. **Page Installation**: Install page in process page table

**Fast Path (Page in Cache)**:
```c
folio = filemap_get_folio(mapping, index);
if (likely(!IS_ERR(folio))) {
    /* Page found in cache */
    if (!(vmf->flags & FAULT_FLAG_TRIED))
        fpin = do_async_mmap_readahead(vmf, folio);
    
    /* Handle page locking and validation */
    if (!lock_folio_maybe_drop_mmap(vmf, folio, &fpin))
        goto out_retry;
}
```

**Slow Path (Page Not in Cache)**:
```c
else {
    /* Major page fault */
    count_vm_event(PGMAJFAULT);
    count_memcg_event_mm(vmf->vma->vm_mm, PGMAJFAULT);
    ret = VM_FAULT_MAJOR;
    
    /* Synchronous readahead */
    fpin = do_sync_mmap_readahead(vmf);
    
    /* Allocate and add page to cache */
    folio = __filemap_get_folio(mapping, index,
                              FGP_CREAT|FGP_FOR_MMAP,
                              vmf->gfp_mask);
}
```

#### `filemap_fault_recheck_pte_none()` - PTE Validation
**Purpose**: Validate that PTE is still none after acquiring locks

**Race Condition Prevention**:
- **PTE Consistency**: Ensure PTE hasn't changed during fault processing
- **Lock Ordering**: Maintain proper lock ordering
- **Retry Logic**: Handle concurrent modifications gracefully

### Readahead Implementation

#### Readahead Strategy Types
```c
/* Readahead patterns */
enum readahead_type {
    RA_PATTERN_INITIAL,     /* Initial readahead */
    RA_PATTERN_SEQUENTIAL,  /* Sequential access pattern */
    RA_PATTERN_RANDOM,      /* Random access pattern */
    RA_PATTERN_CONTEXT,     /* Context-based readahead */
    RA_PATTERN_MMAP,        /* Memory-mapped file readahead */
};
```

#### `do_sync_mmap_readahead()` - Synchronous Readahead
**Purpose**: Perform synchronous readahead for memory-mapped files

**Synchronous Readahead Features**:
- **Immediate Need**: Readahead for immediate page fault
- **Pattern Detection**: Detect access patterns for optimization
- **Resource Management**: Balance readahead with system resources
- **NUMA Awareness**: Consider NUMA topology in readahead decisions

#### `do_async_mmap_readahead()` - Asynchronous Readahead
**Purpose**: Perform asynchronous readahead to improve future performance

**Asynchronous Readahead Benefits**:
- **Performance Optimization**: Improve future access performance
- **Background Processing**: Non-blocking readahead operation
- **Adaptive Algorithms**: Adapt to changing access patterns
- **System Load Balancing**: Balance with overall system load

### Memory Mapping Operations

#### `filemap_map_pages()` - Batch Page Mapping
**Purpose**: Map multiple pages at once for improved performance

**Batch Mapping Process**:
1. **Range Determination**: Determine optimal mapping range
2. **Page Validation**: Validate pages in cache
3. **Batch Installation**: Install multiple pages simultaneously
4. **TLB Optimization**: Optimize TLB usage patterns

#### Memory Mapping Optimizations
- **Fault-Around**: Map nearby pages during page fault
- **Prefaulting**: Proactively map likely-to-be-accessed pages
- **Huge Page Promotion**: Promote to transparent huge pages when beneficial
- **NUMA Optimization**: Consider NUMA locality in mapping decisions

### Page Cache Lookup and Management

#### `filemap_get_folio()` - Primary Cache Lookup
```c
struct folio *filemap_get_folio(struct address_space *mapping, pgoff_t index)
```

**Lookup Features**:
- **XArray Search**: Efficient radix tree search
- **Reference Management**: Proper reference counting
- **Lock-free Operation**: Optimized for concurrent access
- **Error Handling**: Comprehensive error code handling

#### `__filemap_get_folio()` - Extended Cache Operations
```c
struct folio *__filemap_get_folio(struct address_space *mapping,
                                 pgoff_t index, fgf_t fgp_flags,
                                 gfp_t gfp)
```

**Extended Operation Flags**:
```c
#define FGP_ACCESSED    0x00000001  /* Mark page accessed */
#define FGP_LOCK        0x00000002  /* Lock the page */
#define FGP_CREAT       0x00000004  /* Create page if missing */
#define FGP_WRITE       0x00000008  /* Intent to write */
#define FGP_NOFS        0x00000010  /* No filesystem calls */
#define FGP_NOWAIT      0x00000020  /* Don't wait for lock */
#define FGP_FOR_MMAP    0x00000040  /* Memory mapping context */
#define FGP_HEAD        0x00000080  /* Return head page */
#define FGP_ENTRY       0x00000100  /* Return any entry */
#define FGP_STABLE      0x00000200  /* Stable page required */
```

### Writeback and Synchronization

#### Page Writeback Management
- **Dirty Page Tracking**: Track modified pages
- **Writeback Coordination**: Coordinate with writeback subsystem
- **Congestion Management**: Handle storage device congestion
- **Error Handling**: Handle writeback errors gracefully

#### `filemap_range_needs_writeback()` - Writeback Decision
**Purpose**: Determine if a range needs writeback before read

**Writeback Scenarios**:
- **O_DIRECT I/O**: Direct I/O requires clean cache
- **Memory Pressure**: Reclaim requires clean pages
- **Sync Operations**: Explicit synchronization requests
- **Integrity Requirements**: Data integrity operations

### Lock Ordering and Synchronization

#### Complex Lock Hierarchy
```c
/*
 * Lock ordering:
 *
 *  ->i_mmap_rwsem         (truncate_pagecache)
 *    ->private_lock       (__free_pte->block_dirty_folio)
 *      ->swap_lock        (exclusive_swap_page, others)
 *        ->i_pages lock
 *
 *  ->i_rwsem
 *    ->invalidate_lock    (acquired by fs in truncate path)
 *      ->i_mmap_rwsem     (truncate->unmap_mapping_range)
 *
 *  ->mmap_lock
 *    ->i_mmap_rwsem
 *      ->page_table_lock or pte_lock  (various, mainly in memory.c)
 *        ->i_pages lock   (arch-dependent flush_dcache_mmap_lock)
 */
```

**Synchronization Features**:
- **Deadlock Prevention**: Carefully ordered lock acquisition
- **RCU Integration**: Read-copy-update for scalable reads
- **Invalidation Protection**: Prevent races during truncation
- **Memory Barrier Coordination**: Proper memory ordering

## Advanced Features

### Transparent Huge Page Integration

#### Large Folio Support
```c
/* Huge page accounting */
if (folio_test_pmd_mappable(folio)) {
    __lruvec_stat_mod_folio(folio, NR_FILE_THPS, nr);
    filemap_nr_thps_dec(mapping);
}
```

**THP Features**:
- **Automatic Promotion**: Promote regular pages to huge pages
- **Fallback Handling**: Graceful fallback to regular pages
- **Split Operations**: Split huge pages when necessary
- **Accounting**: Accurate huge page accounting

### NUMA Optimization

#### NUMA-Aware Page Allocation
- **Local Allocation**: Prefer local NUMA node allocation
- **Migration Support**: Support page migration between nodes
- **Access Tracking**: Track cross-node access patterns
- **Balancing Integration**: Integration with NUMA balancing

### Memory Cgroup Integration

#### Cgroup Accounting and Limits
- **Memory Accounting**: Accurate per-cgroup memory accounting
- **Limit Enforcement**: Enforce memory limits
- **Reclaim Coordination**: Coordinate with cgroup reclaim
- **Statistics Reporting**: Detailed cgroup statistics

### Workingset Detection

#### Shadow Entry Management
```c
/* Shadow entries for workingset detection */
if (shadowp)
    *shadowp = old;  /* Preserve shadow for workingset tracking */
```

**Workingset Features**:
- **Access Tracking**: Track page access patterns
- **Refault Distance**: Measure refault distances
- **Protection Decision**: Decide which pages to protect
- **LRU Optimization**: Optimize LRU list management

### Error Handling and Recovery

#### Error Injection Support
```c
ALLOW_ERROR_INJECTION(__filemap_add_folio, ERRNO);
```

**Error Handling Features**:
- **Graceful Degradation**: Handle errors without system failure
- **Error Propagation**: Proper error code propagation
- **Resource Cleanup**: Clean up resources on error paths
- **Testing Support**: Support for error injection testing

#### Hardware Error Integration
- **Memory Poisoning**: Handle hardware memory errors
- **Error Isolation**: Isolate corrupted pages
- **Recovery Mechanisms**: Attempt recovery when possible
- **Notification Systems**: Notify applications of errors

### Performance Monitoring

#### Statistics and Tracing
```c
trace_mm_filemap_fault(mapping, index);
trace_mm_filemap_add_to_page_cache(folio);
```

**Monitoring Features**:
- **Performance Counters**: Detailed performance statistics
- **Tracing Integration**: Integration with kernel tracing
- **Debugging Support**: Comprehensive debugging information
- **Profiling Support**: Support for performance profiling

## Integration Points

### Virtual Memory Manager Integration
- **Page Fault Handling**: Primary interface for file page faults
- **Memory Mapping**: Support for mmap() system call
- **Copy-on-Write**: Integration with COW semantics
- **Shared Memory**: Support for shared memory mappings

### File System Integration
- **Address Space Operations**: Generic interface for file systems
- **Readpage/Writepage**: File system I/O operations
- **Truncation**: Support for file truncation
- **Direct I/O**: Integration with direct I/O operations

### Block Layer Integration
- **I/O Scheduling**: Coordinate with I/O schedulers
- **Readahead**: Integration with block layer readahead
- **Error Handling**: Handle block layer errors
- **Congestion Control**: Coordinate with block device congestion

### Memory Reclaim Integration
- **LRU Management**: Integration with LRU lists
- **Page Reclaim**: Support for memory reclaim
- **Writeback Coordination**: Coordinate with writeback
- **Slab Shrinking**: Support for slab cache shrinking

This comprehensive file mapping implementation provides the foundation for file-backed memory operations in Linux, enabling efficient, scalable, and reliable access to file data through both traditional I/O interfaces and memory mapping, while supporting modern features like huge pages, NUMA optimization, and advanced caching algorithms.