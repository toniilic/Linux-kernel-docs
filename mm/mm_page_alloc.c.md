# Linux Kernel Memory Management: page_alloc.c

## Overview

The `page_alloc.c` file implements the core page allocator for the Linux kernel, featuring the **buddy system algorithm** for managing physical memory allocation and deallocation. This is the fundamental building block of the kernel's memory management subsystem, responsible for allocating and freeing physical pages.

## Core Functionality

### 1. Buddy System Algorithm

The buddy system is a memory allocation algorithm that maintains free memory in power-of-2 sized blocks. Key characteristics:

- **Coalescing**: Adjacent free blocks of the same size are merged into larger blocks
- **Splitting**: Large blocks are split into smaller buddies when needed
- **Anti-fragmentation**: Reduces external fragmentation through intelligent merging

#### Main Functions
- `__free_one_page()`: Core buddy system freeing function with coalescing
- `__rmqueue_smallest()`: Allocates smallest available block of requested order
- `expand()`: Splits larger blocks into smaller ones

### 2. Zone-Based Memory Management

Memory is organized into zones based on addressing constraints and characteristics:

```c
// Zone types defined in the system
#ifdef CONFIG_ZONE_DMA
    "DMA",        // ISA DMA-capable memory (< 16MB)
#endif
#ifdef CONFIG_ZONE_DMA32
    "DMA32",      // 32-bit DMA-capable memory (< 4GB)
#endif
    "Normal",     // Normal memory
#ifdef CONFIG_HIGHMEM
    "HighMem",    // High memory (on 32-bit systems)
#endif
    "Movable",    // Movable pages for defragmentation
```

#### Zone Structure Components
- **Free areas**: Arrays of free page lists organized by order (0-MAX_PAGE_ORDER)
- **Watermarks**: Low, high, and minimum memory thresholds
- **Per-CPU pagesets**: Hot/cold page caches for fast allocation

### 3. Migration Types and Anti-Fragmentation

Pages are classified by mobility to reduce fragmentation:

```c
enum migratetype {
    MIGRATE_UNMOVABLE,    // Pages that cannot be moved
    MIGRATE_MOVABLE,      // User pages that can be moved
    MIGRATE_RECLAIMABLE,  // Pages that can be reclaimed
    MIGRATE_HIGHATOMIC,   // High-order atomic allocations
    MIGRATE_CMA,          // Contiguous Memory Allocator
    MIGRATE_ISOLATE,      // Isolated pages
};
```

## Key Data Structures

### 1. Per-CPU Pagesets (PCP)

Fast allocation cache to reduce zone lock contention:

```c
struct per_cpu_pages {
    spinlock_t lock;           // Protection for the PCP
    int count;                 // Number of pages in the lists
    int high;                  // High watermark
    int batch;                 // Batch size for bulk operations
    short free_factor;         // Batch factor for freeing
    short expire;              // Aging counter
    struct list_head lists[];  // Per-migratetype lists
};
```

### 2. Free Area Structure

Buddy system organization per order:

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];  // Free lists per migrate type
    unsigned long nr_free;                       // Number of free pages
};
```

### 3. Zone Structure

Central zone management structure containing:
- Free area arrays for buddy system
- Watermark levels (min, low, high)
- Per-CPU pagesets
- Zone statistics and locks
- Memory pressure and reclaim state

## Memory Allocation Paths

### 1. Fast Path Allocation (`get_page_from_freelist`)

Direct allocation without expensive operations:
1. **Zone iteration**: Try preferred zones first
2. **Watermark check**: Ensure sufficient free memory
3. **PCP allocation**: Try per-CPU cache first
4. **Buddy allocation**: Fall back to buddy system
5. **Preparation**: Clear pages, set references

### 2. Slow Path Allocation (`__alloc_pages_slowpath`)

Complex allocation when fast path fails:
1. **Reclaim**: Try to free memory through page reclaim
2. **Compaction**: Compact memory to create larger free blocks
3. **OOM handling**: Trigger out-of-memory killer if necessary
4. **Retry logic**: Multiple retry attempts with different strategies

### 3. Allocation Context (`struct alloc_context`)

Maintains allocation state across retry attempts:
- Zone preferences and constraints
- Migration type preferences
- Allocation flags and priorities
- NUMA node policies

## Memory Deallocation

### Free Page Flow

1. **Validation**: Check page state and flags
2. **Zone determination**: Identify target zone
3. **Buddy coalescing**: Merge with adjacent free buddies
4. **List management**: Add to appropriate free list
5. **Accounting**: Update zone statistics

#### Key Functions
- `__free_pages()`: Main public interface
- `__free_one_page()`: Core buddy system freeing
- `free_pcppages_bulk()`: Bulk PCP freeing

## Performance Optimizations

### 1. Per-CPU Caches

- **Hot/Cold pages**: Maintain temperature-aware page lists
- **Batch operations**: Reduce lock contention through batching
- **Prefetching**: Optimize cache line usage

### 2. Memory Ordering

- **Page shuffling**: Randomize free page selection for security
- **Allocation ordering**: Prefer specific memory regions
- **Merge prediction**: Optimize buddy merge likelihood

### 3. Scalability Features

- **Zone locks**: Fine-grained locking per zone
- **PCP parallelism**: Per-CPU structures reduce contention
- **Bulk operations**: Efficient multi-page allocation/deallocation

## Watermark Management

### Watermark Levels

- **MIN**: Absolute minimum for kernel allocations
- **LOW**: Triggers background reclaim (kswapd)
- **HIGH**: Reclaim stops when reached

### Dynamic Adjustment

```c
// Watermark calculation factors
static int watermark_boost_factor = 15000;  // Boost amount
static int watermark_scale_factor = 10;     // Scaling factor
int min_free_kbytes = 1024;                 // Minimum free memory
```

## Integration with Other Subsystems

### 1. Memory Compaction

- **Capture control**: Intercept allocations during compaction
- **Migration assistance**: Support for page migration
- **Fragmentation reduction**: Coordinate with compaction algorithms

### 2. Memory Control Groups (cgroups)

- **Accounting**: Track per-cgroup allocations
- **Limits enforcement**: Respect cgroup memory limits
- **Statistics**: Provide cgroup-aware memory statistics

### 3. NUMA Support

- **Node preferences**: Allocate from preferred NUMA nodes
- **Fallback policies**: Handle cross-node allocation
- **Statistics tracking**: Per-node allocation accounting

## Security and Isolation

### 1. Page Poisoning

- **Free page poisoning**: Fill freed pages with poison patterns
- **Allocation clearing**: Zero pages on allocation
- **Debug support**: Detect use-after-free bugs

### 2. Access Control

- **GFP flag validation**: Enforce allocation constraints
- **Zone restrictions**: Prevent inappropriate zone access
- **Capability checking**: Verify allocation permissions

## Error Handling and Debugging

### 1. Bad Page Detection

```c
static void bad_page(struct page *page, const char *reason)
{
    // Rate-limited bad page reporting
    // Stack trace generation
    // Page state dumping
    // System taint marking
}
```

### 2. Allocation Failure Handling

- **Retry mechanisms**: Multiple allocation strategies
- **Fallback policies**: Alternative allocation methods
- **OOM notification**: Trigger memory reclaim or OOM killer
- **Diagnostic output**: Detailed failure analysis

## Configuration and Tuning

### Kernel Configuration Options

- `CONFIG_COMPACTION`: Memory compaction support
- `CONFIG_CMA`: Contiguous Memory Allocator
- `CONFIG_NUMA`: Non-Uniform Memory Access support
- `CONFIG_HIGHMEM`: High memory support (32-bit)

### Runtime Tunables

- `/proc/sys/vm/min_free_kbytes`: Minimum free memory
- `/proc/sys/vm/watermark_scale_factor`: Watermark scaling
- `/proc/sys/vm/watermark_boost_factor`: Temporary watermark boost

## Performance Characteristics

### Time Complexity

- **Fast path allocation**: O(1) for PCP hits
- **Buddy allocation**: O(log n) for buddy search
- **Coalescing**: O(log n) for merge operations

### Space Complexity

- **Metadata overhead**: ~2% of total memory for page structures
- **Free list overhead**: Minimal linked list storage
- **PCP memory**: Small per-CPU cache overhead

## Common Issues and Solutions

### 1. Memory Fragmentation

**Symptoms**: High-order allocation failures despite sufficient free memory
**Solutions**: 
- Enable memory compaction
- Adjust migrate type policies
- Tune PCP batch sizes

### 2. Lock Contention

**Symptoms**: High CPU usage in allocation paths
**Solutions**:
- Increase PCP batch sizes
- Optimize allocation patterns
- Consider NUMA topology

### 3. Allocation Latency

**Symptoms**: Slow allocation performance
**Solutions**:
- Adjust watermarks for background reclaim
- Optimize zone ordering
- Reduce allocation order when possible

This page allocator serves as the foundation for all kernel memory management, providing efficient, scalable, and reliable physical memory allocation services to the entire Linux kernel.