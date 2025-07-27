# Linux Kernel Memory Management: slub.c

## Overview

The `slub.c` file implements the **SLUB (Simple List of Unqueued Blocks) allocator**, which is the default slab allocator in Linux. SLUB focuses on **simplicity and performance** by reducing overhead and improving cache line utilization compared to the traditional SLAB allocator. It provides efficient allocation and deallocation of fixed-size objects with minimal metadata overhead.

## Core Design Philosophy

### 1. Simplicity over Complexity
- **Minimal metadata**: Reduced per-slab overhead
- **Simplified data structures**: Fewer lists and pointers
- **Direct allocation**: Less indirection in fast paths
- **Lock-free operations**: Extensive use of cmpxchg operations

### 2. Performance Focus
- **CPU cache efficiency**: Better cache line utilization
- **Fast path optimization**: Lockless allocation/deallocation
- **Per-CPU structures**: Reduced contention
- **Atomic operations**: Lock-free when possible

## Key Data Structures

### 1. Per-CPU Cache Structure

```c
struct kmem_cache_cpu {
    union {
        struct {
            void **freelist;        // Pointer to next available object
            unsigned long tid;      // Transaction ID for lockless ops
        };
        freelist_aba_t freelist_tid; // Combined for atomic updates
    };
    struct slab *slab;             // Current slab being allocated from
    struct slab *partial;          // List of partially filled slabs
    local_lock_t lock;             // Protects slow path operations
    unsigned int stat[NR_SLUB_STAT_ITEMS]; // Performance statistics
};
```

#### Key Features
- **Transaction ID**: Prevents ABA problems in lockless operations
- **Freelist pointer**: Direct access to next available object
- **Current slab**: Active slab for allocations
- **Partial slabs**: Per-CPU partial slab cache for efficiency

### 2. Per-Node Structure

```c
struct kmem_cache_node {
    spinlock_t list_lock;          // Protects node lists
    unsigned long nr_partial;      // Number of partial slabs
    struct list_head partial;      // Partial slabs list
    atomic_long_t nr_slabs;        // Total number of slabs
    atomic_long_t total_objects;   // Total objects in all slabs
    struct list_head full;         // Full slabs (debug only)
};
```

### 3. Slab State Management

```c
// Slab states in SLUB
enum slab_state {
    FROZEN,         // CPU slab, exempt from list management
    PARTIAL,        // Has free objects, on partial list  
    FULL,           // No free objects
    EMPTY           // All objects free, candidate for release
};
```

## Allocation Algorithm

### 1. Fast Path Allocation (`__slab_alloc_node`)

The lockless fast path for optimal performance:

```c
static __always_inline void *__slab_alloc_node(struct kmem_cache *s,
        gfp_t gfpflags, int node, unsigned long addr, size_t orig_size)
{
    struct kmem_cache_cpu *c;
    struct slab *slab;
    unsigned long tid;
    void *object;

redo:
    // Get per-CPU structure and transaction ID
    c = raw_cpu_ptr(s->cpu_slab);
    tid = READ_ONCE(c->tid);
    barrier();

    object = c->freelist;
    slab = c->slab;

    // Fast path: lockless allocation
    if (likely(object && slab && node_match(slab, node))) {
        void *next_object = get_freepointer_safe(s, object);
        
        // Atomic update of freelist and tid
        if (likely(__update_cpu_freelist_fast(s, object, next_object, tid))) {
            prefetch_freepointer(s, next_object);
            return object;
        }
        goto redo; // CAS failed, retry
    }
    
    // Slow path fallback
    return __slab_alloc(s, gfpflags, node, addr, c, orig_size);
}
```

#### Fast Path Characteristics
- **Lockless operation**: Uses compare-and-swap (CAS)
- **Transaction ID**: Prevents ABA problems
- **CPU affinity**: Allocates from current CPU's slab
- **Prefetching**: Improves cache performance
- **Retry logic**: Handles race conditions gracefully

### 2. Slow Path Allocation (`__slab_alloc`)

Handles complex cases requiring locks or slab allocation:

1. **New slab acquisition**: Get slab from partial lists or allocate new
2. **Node management**: Handle cross-node allocations
3. **Slab state transitions**: Move slabs between lists
4. **Error handling**: OOM conditions and fallbacks

### 3. Allocation Context Management

```c
// Allocation flow with hooks
void *slab_alloc_node(struct kmem_cache *s, gfp_t gfpflags, 
                     int node, unsigned long addr, size_t orig_size)
{
    void *object;
    
    // Pre-allocation hooks
    s = slab_pre_alloc_hook(s, gfpflags);
    if (unlikely(!s))
        return NULL;
    
    // KFENCE integration
    object = kfence_alloc(s, orig_size, gfpflags);
    if (unlikely(object))
        goto out;
    
    // Core allocation
    object = __slab_alloc_node(s, gfpflags, node, addr, orig_size);
    
    // Object initialization
    maybe_wipe_obj_freeptr(s, object);

out:
    // Post-allocation hooks (KASAN, memcg, etc.)
    slab_post_alloc_hook(s, lru, gfpflags, 1, &object, init, orig_size);
    return object;
}
```

## Deallocation Algorithm

### 1. Fast Path Deallocation (`do_slab_free`)

Lockless freeing to per-CPU cache:

```c
static void do_slab_free(struct kmem_cache *s, struct slab *slab,
                        void *head, void *tail, int cnt, unsigned long addr)
{
    struct kmem_cache_cpu *c;
    unsigned long tid;
    void **freelist;

redo:
    c = raw_cpu_ptr(s->cpu_slab);
    tid = READ_ONCE(c->tid);
    barrier();

    // Check if freeing to current CPU slab
    if (likely(slab == c->slab)) {
        freelist = READ_ONCE(c->freelist);
        set_freepointer(s, tail, freelist);
        
        // Atomic freelist update
        if (likely(__update_cpu_freelist_fast(s, freelist, head, tid))) {
            return; // Success
        }
        goto redo; // CAS failed, retry
    }
    
    // Slow path: different slab
    __slab_free(s, slab, head, tail, cnt, addr);
}
```

### 2. Slow Path Deallocation (`__slab_free`)

Handles complex freeing scenarios:

1. **Slab state transitions**: Empty → partial → full
2. **List management**: Move slabs between node lists  
3. **Coalescing**: Return empty slabs to page allocator
4. **Cross-CPU freeing**: Handle remote CPU allocations

### 3. Bulk Operations

SLUB supports efficient bulk allocation and deallocation:
- **Bulk allocation**: Allocate multiple objects in one operation
- **Bulk freeing**: Free multiple objects together
- **Reduced overhead**: Amortize lock acquisition costs
- **Cache efficiency**: Better memory locality

## Lock Hierarchy and Synchronization

### 1. Lock Ordering

```c
/*
 * Lock order:
 *   1. slab_mutex (Global Mutex)
 *   2. node->list_lock (Spinlock)  
 *   3. kmem_cache->cpu_slab->lock (Local lock)
 *   4. slab_lock(slab) (Only on some arches)
 *   5. object_map_lock (Only for debugging)
 */
```

### 2. Lockless Fast Path

- **Compare-and-swap**: Atomic freelist updates
- **Transaction IDs**: Prevent ABA problems
- **Memory barriers**: Ensure proper ordering
- **CPU migration handling**: Detect and handle CPU changes

### 3. Slab States and Synchronization

```c
// Slab state meanings in SLUB
// - node partial slab: PG_Workingset && !frozen
// - cpu partial slab: !PG_Workingset && !frozen  
// - cpu slab: !PG_Workingset && frozen
// - full slab: !PG_Workingset && !frozen
```

## Memory Layout and Optimization

### 1. Object Layout

```c
// Object organization in SLUB
struct slab_object {
    // User data area
    char data[object_size];
    
    // Free pointer (when object is free)
    void *next_free;  // Stored at offset s->offset
    
    // Debug information (if enabled)
    struct track alloc_track;
    struct track free_track;
};
```

### 2. Freelist Encoding

Security hardening through pointer obfuscation:

```c
static inline freeptr_t freelist_ptr_encode(const struct kmem_cache *s,
                                           void *ptr, unsigned long ptr_addr)
{
    unsigned long encoded;
#ifdef CONFIG_SLAB_FREELIST_HARDENED
    encoded = (unsigned long)ptr ^ s->random ^ swab(ptr_addr);
#else
    encoded = (unsigned long)ptr;
#endif
    return (freeptr_t){.v = encoded};
}
```

### 3. Cache Line Optimization

- **Alignment**: Objects aligned to cache line boundaries
- **Coloring**: Offset objects to reduce conflicts
- **Prefetching**: Software prefetch of next objects
- **False sharing avoidance**: Careful structure layout

## Performance Features

### 1. Per-CPU Partial Slabs

```c
#ifdef CONFIG_SLUB_CPU_PARTIAL
// CPU partial slabs for improved performance
struct slab *partial;  // Per-CPU partial slab list

// Benefits:
// - Reduced node lock contention
// - Better cache locality  
// - Faster allocation from partial slabs
// - NUMA-aware allocation
#endif
```

### 2. Statistics and Monitoring

```c
enum stat_item {
    ALLOC_FASTPATH,     // Fast path allocations
    ALLOC_SLOWPATH,     // Slow path allocations  
    FREE_FASTPATH,      // Fast path frees
    FREE_SLOWPATH,      // Slow path frees
    ALLOC_FROM_PARTIAL, // Allocations from partial list
    CMPXCHG_DOUBLE_FAIL,// CAS failure count
    // ... many more statistics
};
```

### 3. NUMA Optimization

- **Node-local allocation**: Prefer local memory
- **Cross-node fallback**: Handle memory pressure
- **Strict NUMA mode**: Enforce node locality when requested
- **Memory policy integration**: Support for mempolicy

## Debug and Validation

### 1. Debug Features

```c
#ifdef CONFIG_SLUB_DEBUG
// Debug flags for validation
#define DEBUG_DEFAULT_FLAGS (SLAB_CONSISTENCY_CHECKS | SLAB_RED_ZONE | \
                            SLAB_POISON | SLAB_STORE_USER)

// Object tracking
struct track {
    unsigned long addr;     // Allocation/free address
    depot_stack_handle_t handle; // Stack trace
    int cpu;               // CPU number
    int pid;               // Process ID  
    unsigned long when;    // Timestamp
};
#endif
```

### 2. Validation Mechanisms

- **Red zones**: Detect buffer overflows
- **Poisoning**: Fill freed objects with patterns
- **Object tracking**: Record allocation/free stack traces
- **Consistency checks**: Validate internal structures
- **KASAN integration**: Address sanitizer support

## Integration Points

### 1. Memory Control Groups

```c
// Memcg integration for accounting
static void memcg_slab_free_hook(struct kmem_cache *s, struct slab *slab,
                                void **p, int objects)
{
    if (memcg_kmem_online() && !is_root_cache(s))
        __memcg_kmem_uncharge(slab_objcgs(slab), objects);
}
```

### 2. KFENCE Integration

- **Sampling**: Random allocation sampling for debugging
- **Out-of-bounds detection**: Catch buffer overflows
- **Use-after-free detection**: Detect freed object access
- **Stack trace recording**: Capture allocation context

### 3. Security Features

```c
#ifdef CONFIG_SLAB_FREELIST_HARDENED
// Freelist pointer hardening
unsigned long s->random;  // Per-cache random value
// XOR encoding prevents freelist manipulation attacks
#endif
```

## Configuration and Tuning

### 1. Compile-Time Options

- `CONFIG_SLUB`: Enable SLUB allocator
- `CONFIG_SLUB_DEBUG`: Debug support
- `CONFIG_SLUB_CPU_PARTIAL`: Per-CPU partial slabs
- `CONFIG_SLUB_STATS`: Performance statistics
- `CONFIG_SLAB_FREELIST_HARDENED`: Security hardening

### 2. Runtime Parameters

- `slub_debug=`: Enable debugging features
- `slub_min_order=`: Minimum slab order
- `slub_max_order=`: Maximum slab order  
- `slub_min_objects=`: Minimum objects per slab

### 3. Sysfs Interface

- `/sys/kernel/slab/`: Per-cache statistics and tuning
- `/proc/slabinfo`: System-wide slab information
- Cache-specific tunables for optimization

## Error Handling

### 1. Allocation Failures

- **OOM handling**: Graceful degradation under memory pressure
- **Fallback strategies**: Alternative allocation methods
- **Error propagation**: Proper error code handling
- **Recovery mechanisms**: Attempt allocation retry

### 2. Corruption Detection

- **Bad object detection**: Identify corrupted objects
- **Double-free detection**: Prevent duplicate frees
- **Debugging output**: Detailed corruption reports
- **Stack trace capture**: Identify corruption source

## Performance Characteristics

### Time Complexity

- **Fast path allocation**: O(1) lockless operation
- **Slow path allocation**: O(log n) for partial list search
- **Fast path free**: O(1) lockless operation
- **Slow path free**: O(1) with occasional O(log n) list operations

### Space Complexity

- **Metadata overhead**: ~1-2% for object tracking
- **Per-CPU overhead**: Small fixed per-CPU structures
- **Fragmentation**: Reduced compared to buddy allocator
- **Memory efficiency**: Good utilization through object reuse

## Common Usage Patterns

### 1. High-Frequency Allocations

```c
// Typical kernel object allocation
struct my_object *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);
if (obj) {
    // Use object
    kmem_cache_free(my_cache, obj);
}
```

### 2. NUMA-Aware Allocation

```c
// Allocate on specific NUMA node
void *obj = kmem_cache_alloc_node(cache, GFP_KERNEL, node);
```

### 3. Bulk Operations

```c
// Efficient bulk allocation
void **objects = kcalloc(count, sizeof(void *), GFP_KERNEL);
kmem_cache_alloc_bulk(cache, GFP_KERNEL, count, objects);
```

SLUB represents a significant evolution in slab allocation design, emphasizing simplicity, performance, and scalability while maintaining the object caching benefits that make slab allocators effective for kernel memory management.