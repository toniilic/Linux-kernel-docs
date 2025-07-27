# Linux Kernel Memory Management: slab_common.c

## Overview

The `slab_common.c` file implements the **common slab allocator framework** that provides a unified interface for different slab allocator implementations (SLAB, SLUB, SLOB). This serves as the **foundation layer** for kernel object allocation, providing efficient memory management for frequently allocated kernel data structures.

## Core Functionality

### 1. Slab Allocator Framework

The slab allocator is designed for efficient allocation and deallocation of objects of the same size:

- **Object pooling**: Maintains pools of pre-allocated objects
- **Fast allocation**: O(1) allocation/deallocation for hot objects
- **Memory efficiency**: Reduces fragmentation through object reuse
- **Cache-friendly**: Optimizes CPU cache utilization

#### Key Features
- **Multiple implementations**: Supports SLAB, SLUB, and SLOB algorithms
- **Object caching**: Maintains hot/cold object caches
- **Debugging support**: Built-in debugging and profiling capabilities
- **NUMA awareness**: Per-node allocation optimization

### 2. Kmalloc Infrastructure

Provides the `kmalloc()` family of functions for general-purpose kernel memory allocation:

```c
// Kmalloc cache types for different use cases
enum kmalloc_cache_type {
    KMALLOC_NORMAL,      // Normal allocations
    KMALLOC_RECLAIM,     // Reclaimable allocations  
    KMALLOC_DMA,         // DMA-capable memory
    KMALLOC_CGROUP,      // Memory cgroup accounting
    KMALLOC_RANDOM_START, // Randomized caches (security)
    // ... additional random cache types
    NR_KMALLOC_TYPES
};
```

#### Size Classes

Predefined size classes for efficient allocation:
- **Small sizes**: 8, 16, 32, 64, 96, 128, 192 bytes
- **Power-of-2 sizes**: 256, 512, 1k, 2k, 4k, 8k, 16k, 32k, 64k, 128k, 256k, 512k, 1M, 2M
- **Size indexing**: Fast O(1) size-to-cache mapping

### 3. Cache Management and Merging

Intelligent cache merging to reduce memory overhead:

```c
// Flags that prevent cache merging
#define SLAB_NEVER_MERGE (SLAB_RED_ZONE | SLAB_POISON | SLAB_STORE_USER | \
                         SLAB_TRACE | SLAB_TYPESAFE_BY_RCU | SLAB_NOLEAKTRACE | \
                         SLAB_FAILSLAB | SLAB_NO_MERGE)

// Flags that must match for merging
#define SLAB_MERGE_SAME (SLAB_RECLAIM_ACCOUNT | SLAB_CACHE_DMA | \
                        SLAB_CACHE_DMA32 | SLAB_ACCOUNT)
```

## Key Data Structures

### 1. Kmem Cache Structure

Central structure for each slab cache:

```c
struct kmem_cache {
    const char *name;           // Cache name for identification
    unsigned int object_size;   // Size of each object
    unsigned int size;          // Actual allocation size (aligned)
    unsigned int align;         // Required alignment
    slab_flags_t flags;         // Cache behavior flags
    void (*ctor)(void *);       // Object constructor
    int refcount;               // Reference counter
    struct list_head list;      // Global cache list
    // Implementation-specific fields...
};
```

### 2. Slab States

System-wide slab allocator initialization states:

```c
enum slab_state {
    DOWN,       // No slab functionality yet
    PARTIAL,    // SLUB: kmem_cache_node available
    UP,         // Slab caches usable but not all extras yet
    FULL        // Everything is working
};
```

### 3. Kmalloc Size Index Table

Fast size-to-index mapping for small allocations:

```c
u8 kmalloc_size_index[24] = {
    3,  /* 8 bytes  -> index 3 */
    4,  /* 16 bytes -> index 4 */
    5,  /* 24 bytes -> index 5 */
    // ... mapping for sizes up to 192 bytes
};
```

## Major Functions

### 1. Cache Creation

#### `__kmem_cache_create_args()`
Main cache creation interface:
1. **Validation**: Sanity check parameters
2. **Merging**: Find existing compatible cache
3. **Creation**: Create new cache if needed
4. **Registration**: Add to global cache list

#### `create_boot_cache()`
Early boot cache creation before full slab system is available:
- Used for essential kernel structures
- Simple allocation without advanced features
- Marked as non-mergeable initially

### 2. Kmalloc System

#### `create_kmalloc_caches()`
Initialize the kmalloc cache hierarchy:
1. **Size class setup**: Create caches for each size class
2. **Type variation**: Create specialized cache types (DMA, reclaimable, etc.)
3. **Randomization**: Initialize random caches for security
4. **State transition**: Mark slab system as UP

#### `kmalloc_slab()`
Fast cache selection for kmalloc requests:
```c
static inline struct kmem_cache *
kmalloc_slab(size_t size, kmem_buckets *b, gfp_t flags, unsigned long caller)
{
    unsigned int index;
    
    if (!b)
        b = &kmalloc_caches[kmalloc_type(flags, caller)];
    if (size <= 192)
        index = kmalloc_size_index[size_index_elem(size)];
    else
        index = fls(size - 1);  // Find last set bit
        
    return (*b)[index];
}
```

### 3. Cache Management

#### `find_mergeable()`
Find compatible cache for merging:
1. **Compatibility check**: Verify flags and properties
2. **Size requirements**: Ensure size compatibility
3. **Alignment verification**: Check alignment requirements
4. **Best fit selection**: Choose most suitable existing cache

#### `slab_unmergeable()`
Determine if cache can be merged:
- Check for debugging flags
- Verify constructor absence
- Validate reference count
- Check hardened usercopy requirements

## Memory Layout and Alignment

### 1. Alignment Calculation

```c
static unsigned int calculate_alignment(slab_flags_t flags,
                                      unsigned int align, 
                                      unsigned int size)
{
    // Hardware cache alignment for performance
    if (flags & SLAB_HWCACHE_ALIGN) {
        unsigned int ralign = cache_line_size();
        while (size <= ralign / 2)
            ralign /= 2;
        align = max(align, ralign);
    }
    
    // Architecture minimum alignment
    align = max(align, arch_slab_minalign());
    
    // Pointer alignment requirement
    return ALIGN(align, sizeof(void *));
}
```

### 2. Size Optimization

- **Power-of-2 alignment**: Optimizes buddy allocator usage
- **Cache line alignment**: Reduces false sharing
- **Architecture constraints**: Respects platform requirements
- **DMA requirements**: Ensures DMA-safe alignment

## Performance Optimizations

### 1. Fast Path Allocation

- **Size indexing**: O(1) cache selection
- **Per-CPU caches**: Reduced lock contention
- **Object reuse**: Minimizes allocation overhead
- **Prefetching**: Improves cache locality

### 2. Cache Line Optimization

- **SLAB_HWCACHE_ALIGN**: Align objects to cache lines
- **Size tuning**: Optimize object sizes for cache efficiency
- **Layout optimization**: Minimize false sharing

### 3. Memory Efficiency

- **Cache merging**: Reduces number of active caches
- **Size classes**: Balances fragmentation vs. waste
- **Object packing**: Maximizes objects per page

## Security Features

### 1. Random Kmalloc Caches

```c
#ifdef CONFIG_RANDOM_KMALLOC_CACHES
unsigned long random_kmalloc_seed;

// Creates randomized cache layout to hinder exploits
for (type = KMALLOC_RANDOM_START; type <= KMALLOC_RANDOM_END; type++) {
    flags |= SLAB_NO_MERGE;  // Prevent cache merging
    // Create separated caches for security isolation
}
#endif
```

### 2. Object Validation

- **Debugging flags**: Enable object tracking and validation
- **Poison patterns**: Detect use-after-free bugs
- **Red zoning**: Detect buffer overflows
- **Stack traces**: Track allocation/free paths

## Integration Points

### 1. Memory Control Groups (cgroups)

```c
if (IS_ENABLED(CONFIG_MEMCG) && (type == KMALLOC_CGROUP)) {
    if (!mem_cgroup_kmem_disabled()) {
        flags |= SLAB_ACCOUNT;  // Enable cgroup accounting
    }
}
```

### 2. NUMA Support

- **Node-local allocation**: Prefer local memory nodes
- **Cross-node fallback**: Handle memory pressure
- **Topology awareness**: Optimize for NUMA layout

### 3. DMA Integration

```c
if (IS_ENABLED(CONFIG_ZONE_DMA) && (type == KMALLOC_DMA)) {
    flags |= SLAB_CACHE_DMA;  // Ensure DMA-capable memory
}
```

## Debug and Monitoring

### 1. Object Information (`kmem_dump_obj`)

Provides detailed object provenance:
- **Cache identification**: Which cache allocated the object
- **Allocation tracking**: Call stack at allocation time
- **Free tracking**: Call stack at free time (if available)
- **Object state**: Current object status

### 2. Slab Statistics

- **Cache utilization**: Objects allocated vs. available
- **Memory usage**: Total memory consumption per cache
- **Performance metrics**: Allocation/free rates
- **Fragmentation analysis**: Internal vs. external fragmentation

## Boot Process Integration

### 1. Early Initialization

1. **Basic setup**: Initialize core data structures
2. **Bootstrap caches**: Create essential caches for early boot
3. **Architecture setup**: Configure platform-specific parameters
4. **Kmalloc creation**: Establish general-purpose allocation

### 2. State Transitions

- **DOWN → PARTIAL**: Basic slab structures available
- **PARTIAL → UP**: Kmalloc caches operational
- **UP → FULL**: All features and debugging enabled

## Configuration Options

### Compile-Time Configuration

- `CONFIG_SLAB`: Classic SLAB allocator
- `CONFIG_SLUB`: SLUB allocator (default)
- `CONFIG_SLOB`: Simple allocator (deprecated)
- `CONFIG_SLAB_MERGE_DEFAULT`: Enable cache merging by default
- `CONFIG_RANDOM_KMALLOC_CACHES`: Security hardening
- `CONFIG_SLAB_BUCKETS`: Bucket-based allocation API

### Runtime Parameters

- `slab_nomerge`: Disable cache merging
- `slab_merge`: Enable cache merging
- `slub_debug`: Enable SLUB debugging features

## Error Handling

### 1. Allocation Failures

- **Graceful degradation**: Fall back to page allocator
- **Memory pressure**: Coordinate with page reclaim
- **OOM conditions**: Proper error propagation

### 2. Validation and Debugging

- **Parameter validation**: Comprehensive input checking
- **State consistency**: Verify internal state integrity
- **Memory corruption detection**: Multiple validation layers

## Performance Characteristics

### Time Complexity

- **Cache lookup**: O(1) for size-to-cache mapping
- **Object allocation**: O(1) for hot path
- **Cache creation**: O(n) for compatibility checking
- **Merging analysis**: O(n) where n = number of existing caches

### Space Complexity

- **Metadata overhead**: ~1-2% of allocated memory
- **Size class waste**: Controlled internal fragmentation
- **Cache proliferation**: Balanced by merging strategies

This slab common framework provides the essential infrastructure for efficient kernel object allocation, serving as the foundation for all slab allocator implementations while maintaining compatibility and performance across different allocation strategies.