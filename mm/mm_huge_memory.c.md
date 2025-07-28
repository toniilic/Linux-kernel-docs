# Linux Kernel Transparent Huge Pages (`mm/huge_memory.c`)

## Overview

The `mm/huge_memory.c` file implements Linux's Transparent Huge Pages (THP) feature, a critical memory management optimization that automatically uses large pages (typically 2MB) instead of regular 4KB pages for suitable memory allocations. This system significantly reduces TLB pressure, improves memory access performance, and reduces page table overhead for applications with large memory footprints, all without requiring application modifications.

## Core Architecture

### 1. Transparent Huge Page Configuration

**Global Control Flags** - Lines 60-69:
```c
unsigned long transparent_hugepage_flags __read_mostly =
#ifdef CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS
    (1<<TRANSPARENT_HUGEPAGE_FLAG)|
#endif
#ifdef CONFIG_TRANSPARENT_HUGEPAGE_MADVISE
    (1<<TRANSPARENT_HUGEPAGE_REQ_MADV_FLAG)|
#endif
    (1<<TRANSPARENT_HUGEPAGE_DEFRAG_REQ_MADV_FLAG)|
    (1<<TRANSPARENT_HUGEPAGE_DEFRAG_KHUGEPAGED_FLAG)|
    (1<<TRANSPARENT_HUGEPAGE_USE_ZERO_PAGE_FLAG);
```

**Anonymous Memory Order Configuration** - Lines 81-84:
- `huge_anon_orders_always`: Always use these page orders
- `huge_anon_orders_madvise`: Use only with madvise hint
- `huge_anon_orders_inherit`: Inherit from parent process

### 2. Multi-Size THP (MTHP) Support

**Size Configuration** - Lines 101-193:
- **`__thp_vma_allowable_orders()`**: Determines allowable huge page orders for VMA
- **Support Matrix**: Different page sizes for anonymous, file, and special mappings
- **Inheritance Model**: Process-level configuration inherited across fork

**Order Validation**:
- `THP_ORDERS_ALL_ANON`: All supported anonymous memory orders
- `THP_ORDERS_ALL_FILE_DEFAULT`: Default file-backed orders
- `THP_ORDERS_ALL_SPECIAL`: Special mapping orders

### 3. Huge Zero Page Management

**Shared Zero Page** - Lines 78-80:
```c
static atomic_t huge_zero_refcount;
struct folio *huge_zero_folio __read_mostly;
unsigned long huge_zero_pfn __read_mostly = ~0UL;
```

**Purpose**: Provides memory-efficient handling of large zero-filled regions by sharing a single huge zero page across multiple processes.

## Sysfs Configuration Interface

### 1. Per-Size Control Interface

**Anonymous Memory Controls** - Lines 509-551:
- **`/sys/kernel/mm/transparent_hugepage/hugepages-<size>kB/enabled`**
- **Options**: `always`, `inherit`, `madvise`, `never`
- **Locking**: `huge_anon_orders_lock` protects concurrent modifications

**File-backed Memory Controls**:
- **Shmem Integration**: Per-filesystem and per-mount controls
- **Read-only File Support**: `CONFIG_READ_ONLY_THP_FOR_FS`

### 2. Statistics Interface

**MTHP Statistics** - Lines 589-689:
```c
DEFINE_PER_CPU(struct mthp_stat, mthp_stats) = {{{0}}};
```

**Tracked Metrics**:
- `anon_fault_alloc`/`anon_fault_fallback`: Allocation success/failure
- `swpin`/`swpout`: Swap-in/swap-out operations  
- `split`/`split_failed`: Page splitting operations
- `nr_anon`/`nr_anon_partially_mapped`: Current page counts

## Page Fault Handling

### 1. Huge PMD Page Faults

**Write Protection Handling** - Lines 2015-2043:
- **`can_change_pmd_writable()`**: Determines if PMD can be made writable
- **Conditions**: Anonymous exclusive pages, dirty shared pages
- **Integration**: NUMA hinting, userfaultfd, soft-dirty tracking

**NUMA Balancing** - Lines 2046-2120:
- **`do_huge_pmd_numa_page()`**: Handles NUMA hinting page faults
- **Migration Decision**: Based on NUMA topology and access patterns
- **Performance**: Reduces migration overhead for large pages

### 2. Memory Advice Integration

**MADV_FREE Support** - Lines 2126-2194:
- **`madvise_free_huge_pmd()`**: Implements lazy freeing for huge pages
- **Optimization**: Marks pages as lazy-free instead of immediate deallocation
- **Fallback**: Splits page if partial range specified

## Page Splitting Infrastructure

### 1. PMD Splitting

**Core Splitting Function** - Lines 3083-3106:
- **`__split_huge_pmd()`**: Main PMD splitting implementation
- **MMU Notifier Integration**: Coordinates with external memory users
- **Locking**: Careful ordering to prevent races

**Splitting Mechanics** - Lines 3000-3081:
```c
// Key steps in PMD splitting:
1. Withdraw page table from PMD
2. Create individual PTEs for each 4KB page
3. Handle migration entries for swapped pages
4. Update memory management counters
5. Invalidate TLB entries
```

### 2. Automatic Splitting

**VMA Adjustment Integration** - Lines 3131-3145:
- **`vma_adjust_trans_huge()`**: Splits huge pages during VMA operations
- **Address Alignment**: Ensures proper alignment during memory operations
- **Boundary Handling**: Splits at VMA boundaries when necessary

## Memory Reclaim Integration

### 1. Deferred Splitting

**Shrinker Integration** - Lines 71-76:
```c
static struct shrinker *deferred_split_shrinker;
static bool split_underused_thp = true;
```

**Purpose**: Automatically splits underused huge pages during memory pressure to reclaim unused portions.

### 2. Folio Management

**Unmapping Operations** - Lines 3147-3168:
- **`unmap_folio()`**: Removes all mappings from a large folio
- **Migration vs. Unmapping**: Different strategies for anonymous vs. file pages
- **TTU Flags**: Controls splitting behavior during unmapping

## Anonymous Page Discard

### 1. PMD-level Discard

**`__discard_anon_folio_pmd_locked()`** - Lines 3170-3199:
- **Purpose**: Efficiently discards entire PMD-mapped anonymous folios
- **Dirty Page Handling**: Respects VM_DROPPABLE flag for dirty pages
- **GUP-fast Synchronization**: Memory barriers to coordinate with fast GUP

**Reference Count Validation**:
- Ensures safe discard only when no external references exist
- Coordinates with concurrent page table walkers
- Handles race conditions with memory allocation

## Performance Optimizations

### 1. TLB Efficiency

**Large Page Benefits**:
- **Reduced TLB Pressure**: Single TLB entry covers 2MB instead of 512 entries
- **Page Table Overhead**: Reduces page table memory usage
- **Cache Performance**: Better locality for large working sets

### 2. Allocation Strategies

**Order Selection Logic**:
- **Alignment Requirements**: Ensures proper address alignment for huge pages
- **VMA Suitability**: Checks VMA size and characteristics
- **Fragmentation Avoidance**: Intelligent fallback to smaller orders

## Security and Isolation

### 1. Process Isolation

**Credential Checking**:
- **Capability Requirements**: Administrative control over system-wide settings
- **Per-process Configuration**: Process-specific huge page preferences
- **Inheritance Control**: Configurable inheritance across process boundaries

### 2. Memory Protection

**Write Protection Handling**:
- **Anonymous Exclusivity**: Ensures safe write access to anonymous pages
- **Soft-dirty Tracking**: Integration with memory change tracking
- **Userfaultfd Integration**: Support for user-space page fault handling

## Integration Points

### 1. Scheduler Integration

**NUMA Balancing**:
- **Migration Decisions**: Coordinates with NUMA balancer for optimal placement
- **CPU Assignment**: Considers CPU topology in migration decisions
- **Load Distribution**: Balances memory access patterns

### 2. Swap Subsystem

**Huge Page Swapping**:
- **Efficient Swap-out**: Handles entire huge pages as units
- **Swap-in Optimization**: Reconstructs huge pages during swap-in
- **Fragmentation Handling**: Falls back to regular pages when necessary

### 3. Memory Cgroup Integration

**Resource Accounting**:
- **Per-cgroup Statistics**: Tracks huge page usage per control group
- **Limit Enforcement**: Respects memory limits during huge page allocation
- **OOM Integration**: Considers huge pages in out-of-memory decisions

## Configuration and Tuning

### 1. Compile-time Options

**Feature Selection**:
- `CONFIG_TRANSPARENT_HUGEPAGE`: Base THP support
- `CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS`: Always-on mode
- `CONFIG_TRANSPARENT_HUGEPAGE_MADVISE`: Advisory-only mode
- `CONFIG_READ_ONLY_THP_FOR_FS`: File-backed THP support

### 2. Runtime Configuration

**Sysfs Controls**:
- `/sys/kernel/mm/transparent_hugepage/enabled`: Global enable/disable
- `/sys/kernel/mm/transparent_hugepage/defrag`: Defragmentation behavior
- `/sys/kernel/mm/transparent_hugepage/use_zero_page`: Zero page optimization

**Per-size Configuration**:
- Individual control for different huge page sizes
- Statistics collection for performance analysis
- Process inheritance models

## Error Handling and Robustness

### 1. Allocation Failures

**Graceful Fallback**:
- **Order Reduction**: Tries smaller huge page sizes on allocation failure
- **Regular Page Fallback**: Falls back to 4KB pages when necessary
- **Statistics Tracking**: Records fallback events for analysis

### 2. Memory Pressure Response

**Adaptive Behavior**:
- **Automatic Splitting**: Splits underused huge pages during pressure
- **Deferred Operations**: Queues splitting operations for later execution
- **Priority Handling**: Considers allocation urgency in decisions

## Development and Debugging

### 1. Tracing Support

**Trace Points**:
- `CREATE_TRACE_POINTS` for THP events
- Allocation success/failure tracking
- Page splitting and merging events

### 2. Debug Infrastructure

**Validation Checks**:
- `VM_BUG_ON` assertions for critical invariants
- Reference count validation
- Page state consistency checks

The transparent huge pages implementation represents a sophisticated balance between performance optimization and system stability, providing significant memory access improvements while maintaining compatibility with existing applications and memory management semantics. Its multi-layered approach to configuration, statistics, and error handling makes it a robust foundation for high-performance computing workloads.