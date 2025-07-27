# mm/page_alloc.c - Page Frame Allocator Implementation

## Overview

This file implements the core page frame allocator for the Linux kernel memory management subsystem. Originally written by Linus Torvalds and extensively enhanced by numerous contributors, it manages the system's free page lists and provides the fundamental page allocation and deallocation services that underpin all kernel memory management.

## Historical Development

### Key Contributors and Milestones
- **1991-1994**: Original implementation by Linus Torvalds
- **1995**: Swap reorganization by Stephen Tweedie
- **1999**: BIGMEM support by Gerhard Wichert (Siemens AG)
- **1999**: Zoned allocator transformation by Ingo Molnar (Red Hat)
- **1999**: Discontiguous memory support by Kanoj Sarcar (SGI)
- **2000**: Zone balancing by Kanoj Sarcar (SGI)
- **2002**: Per-CPU hot/cold page lists by Martin J. Bligh

### Architecture Evolution
- **Simple Free Lists → Zoned Allocator**: Better NUMA and device constraint handling
- **Single Lists → Per-CPU Lists**: Improved SMP scalability
- **Static → Dynamic**: Support for memory hotplug and heterogeneous systems

## Core Concepts

### Memory Zones
The allocator organizes memory into zones based on hardware constraints:

#### Zone Types
- **ZONE_DMA**: Memory suitable for ISA DMA (< 16MB on x86)
- **ZONE_DMA32**: Memory suitable for 32-bit DMA (< 4GB on x86_64)
- **ZONE_NORMAL**: Normal memory accessible by kernel
- **ZONE_HIGHMEM**: Memory not permanently mapped (32-bit systems)
- **ZONE_MOVABLE**: Memory that can be migrated/reclaimed

#### Zone Characteristics
- **Hardware Constraints**: DMA accessibility requirements
- **Kernel Mapping**: Direct vs. temporary mapping
- **Migration**: Whether pages can be moved
- **Reclaim**: Whether pages can be reclaimed

### Buddy Allocator System

#### Algorithm Principles
- **Power-of-2 Sizes**: Allocations in orders (2^order pages)
- **Buddy Pairing**: Adjacent blocks of same size are buddies
- **Merging**: Free buddies merge into larger blocks
- **Splitting**: Large blocks split for smaller requests

#### Block Orders
- **Order 0**: Single pages (4KB on x86)
- **Order 1**: 2-page blocks (8KB)
- **Order 2**: 4-page blocks (16KB)
- **...up to...**
- **Order 10**: 1024-page blocks (4MB) - maximum order

### Per-CPU Page Lists (PCP)

#### Hot/Cold Lists
- **Hot Pages**: Recently freed, likely in CPU cache
- **Cold Pages**: Not recently used, cache-cold
- **Fast Path**: Per-CPU allocation without locks
- **Batch Operations**: Bulk allocation/freeing for efficiency

#### Benefits
- **Lock Avoidance**: Reduce buddy allocator lock contention
- **Cache Locality**: Improve CPU cache performance
- **Batch Efficiency**: Amortize locking overhead

## Key Data Structures

### `struct zone` - Memory Zone Descriptor
```c
struct zone {
    unsigned long _watermarks[NR_WMARK];  /* Low, min, high watermarks */
    unsigned long nr_reserved_highatomic; /* High-priority atomic reserves */
    
    long lowmem_reserve[MAX_NR_ZONES];    /* Protection from lower zones */
    
    struct per_cpu_pages __percpu *per_cpu_pageset; /* Per-CPU page lists */
    
    struct free_area free_area[NR_PAGE_ORDERS]; /* Buddy allocator lists */
    
    unsigned long zone_start_pfn;         /* First PFN in zone */
    unsigned long spanned_pages;          /* Total pages spanned */
    unsigned long present_pages;          /* Actual pages present */
    unsigned long managed_pages;          /* Pages managed by buddy */
    
    const char *name;                     /* Zone name */
    int initialized;                      /* Zone initialization state */
};
```

### `struct per_cpu_pages` - Per-CPU Page Cache
```c
struct per_cpu_pages {
    spinlock_t lock;              /* Protects list */
    int count;                    /* Number of pages in lists */
    int high;                     /* High watermark */
    int low;                      /* Low watermark */
    int batch;                    /* Batch size for bulk operations */
    u8 flags;                     /* Flags */
    u8 alloc_factor;             /* Allocation factor */
    u8 expire;                    /* Expiration counter */
    
    struct list_head lists[NR_PCP_LISTS]; /* Hot/cold page lists */
};
```

### `struct free_area` - Buddy Allocator Lists
```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES]; /* Free lists by migration type */
    unsigned long nr_free;                      /* Number of free blocks */
};
```

## Core Functions

### `__alloc_pages_noprof()` - Main Allocation Interface
```c
struct page *__alloc_pages_noprof(gfp_t gfp, unsigned int order,
                                 int preferred_nid, nodemask_t *nodemask)
```

**Purpose**: Primary page allocation function used by all kernel subsystems

**Allocation Process**:
1. **Fast Path**: Try per-CPU lists first
2. **Zone Selection**: Choose appropriate zone based on GFP flags
3. **Watermark Check**: Verify zone has sufficient free memory
4. **Slow Path**: Handle complex allocation scenarios
5. **Fallback**: Try alternative zones/strategies if needed

**GFP Flags Processing**:
- **Zone Selection**: `__GFP_DMA`, `__GFP_DMA32`, `__GFP_HIGHMEM`
- **Behavior Control**: `__GFP_WAIT`, `__GFP_IO`, `__GFP_FS`
- **Reclaim Policy**: `__GFP_NORETRY`, `__GFP_RETRY_MAYFAIL`
- **Special Handling**: `__GFP_ATOMIC`, `__GFP_EMERGENCY`

### `get_page_from_freelist()` - Fast Path Allocation
```c
static struct page *get_page_from_freelist(gfp_t gfp_mask, unsigned int order,
                                          int alloc_flags, const struct alloc_context *ac)
```

**Fast Path Strategy**:
1. **Zone Iteration**: Try zones in preference order
2. **Watermark Check**: Ensure sufficient free memory
3. **Per-CPU Lists**: Try per-CPU cache first
4. **Buddy Allocator**: Fall back to buddy system
5. **Migration Type**: Handle page migration types

**Optimization Features**:
- **Zone Preference**: Prefer local NUMA nodes
- **Watermark Awareness**: Respect memory pressure indicators
- **Batch Operations**: Efficient bulk allocation
- **Migration Grouping**: Maintain page migration semantics

### `__alloc_pages_slowpath()` - Complex Allocation Handling
```c
static struct page *__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
                                          struct alloc_context *ac)
```

**Slow Path Scenarios**:
1. **Memory Pressure**: System under memory pressure
2. **Large Orders**: High-order allocations requiring effort
3. **Constrained Zones**: Limited zone availability
4. **Special Requirements**: Atomic or emergency allocations

**Recovery Strategies**:
- **Direct Reclaim**: Reclaim pages directly
- **Compaction**: Defragment memory for large orders
- **OOM Handling**: Out-of-memory killer invocation
- **Retry Logic**: Sophisticated retry mechanisms

### Memory Freeing Functions

#### `__free_pages()` - Main Deallocation Interface
```c
void __free_pages(struct page *page, unsigned int order)
```

**Deallocation Process**:
1. **Reference Check**: Verify page can be freed
2. **Order Validation**: Check allocation order
3. **Fast Path**: Use per-CPU lists for single pages
4. **Buddy Return**: Return to buddy allocator
5. **Merge Operation**: Merge with buddy blocks

#### `__free_one_page()` - Buddy System Integration
```c
static inline void __free_one_page(struct page *page, unsigned long pfn,
                                  struct zone *zone, unsigned int order,
                                  int migratetype, fpi_t fpi_flags)
```

**Buddy Merging Algorithm**:
1. **Buddy Location**: Calculate buddy block address
2. **Merge Check**: Verify buddy is free and same order
3. **Iterative Merging**: Merge up to maximum order
4. **List Management**: Update free lists appropriately
5. **Accounting**: Update zone statistics

### Per-CPU Page Lists Management

#### `free_pcppages_bulk()` - Bulk PCP Operations
```c
static void free_pcppages_bulk(struct zone *zone, int count,
                              struct per_cpu_pages *pcp,
                              int pindex)
```

**Bulk Operations**:
- **Batch Processing**: Process multiple pages efficiently
- **Lock Amortization**: Reduce buddy allocator lock overhead
- **Cache Optimization**: Maintain cache locality
- **Load Balancing**: Distribute pages across lists

#### PCP Tuning and Management
```c
static void pcp_update_high_batch(struct zone *zone);
```

**Adaptive Tuning**:
- **Dynamic Sizing**: Adjust PCP sizes based on usage
- **Memory Pressure**: Reduce cache size under pressure
- **Performance Optimization**: Balance speed vs. memory usage
- **NUMA Awareness**: Consider NUMA topology

## Memory Watermarks and Pressure Handling

### Watermark Types
- **WMARK_MIN**: Minimum free memory threshold
- **WMARK_LOW**: Trigger background reclaim
- **WMARK_HIGH**: Stop background reclaim
- **WMARK_PROMO**: Promotion threshold for different zones

### Pressure Response
```c
static bool zone_watermark_fast(struct zone *z, unsigned int order,
                               unsigned long mark, int highest_zoneidx,
                               unsigned int alloc_flags, gfp_t gfp_mask)
```

**Watermark Checking**:
1. **Quick Assessment**: Fast watermark evaluation
2. **Order Consideration**: Account for allocation size
3. **Reserve Handling**: Handle various reserve types
4. **Pressure Detection**: Identify memory pressure conditions

## NUMA Optimization

### Node Selection
```c
static struct page *__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
                                          int preferred_nid, nodemask_t *nodemask)
```

**NUMA-Aware Allocation**:
- **Local Preference**: Prefer local NUMA node
- **Fallback Strategy**: Use distance-based fallback
- **Bandwidth Consideration**: Consider memory bandwidth
- **Load Balancing**: Balance across nodes

### Zone Ordering
- **Distance-Based**: Order zones by NUMA distance
- **Performance-Oriented**: Optimize for access patterns
- **Policy Compliance**: Respect memory policies

## Migration Types and Anti-Fragmentation

### Migration Types
- **MIGRATE_UNMOVABLE**: Kernel allocations that cannot move
- **MIGRATE_MOVABLE**: User pages that can be migrated
- **MIGRATE_RECLAIMABLE**: Pages that can be reclaimed
- **MIGRATE_PCPTYPES**: Types suitable for PCP lists
- **MIGRATE_HIGHATOMIC**: High-priority atomic allocations
- **MIGRATE_CMA**: Contiguous Memory Allocator pages
- **MIGRATE_ISOLATE**: Isolated pages for memory hotplug

### Anti-Fragmentation Strategy
```c
static int fallbacks[MIGRATE_TYPES][MIGRATE_PCPTYPES + 1];
```

**Fragmentation Prevention**:
1. **Type Grouping**: Group pages by mobility
2. **Fallback Order**: Minimize fragmentation during fallbacks
3. **Block Stealing**: Steal entire pageblocks when needed
4. **Compaction Integration**: Work with memory compaction

## Memory Hotplug Integration

### Online/Offline Operations
```c
void __meminit __free_pages_core(struct page *page, unsigned int order,
                                enum meminit_context context)
```

**Hotplug Support**:
- **Memory Addition**: Integrate new memory into allocator
- **Memory Removal**: Safely remove memory regions
- **Zone Management**: Handle zone boundary changes
- **Migration Coordination**: Coordinate with page migration

## Security and Debugging Features

### Page Poisoning and Debugging
```c
static void prep_new_page(struct page *page, unsigned int order,
                         gfp_t gfp_flags, unsigned int alloc_flags)
```

**Security Features**:
- **Page Clearing**: Clear sensitive data from pages
- **Poison Patterns**: Detect use-after-free bugs
- **Boundary Checking**: Detect buffer overflows
- **Allocation Tracking**: Track allocation patterns

### Memory Error Detection
- **KASAN Integration**: AddressSanitizer support
- **KMSAN Integration**: MemorySanitizer support
- **Page Table Checking**: Detect page table corruptions
- **Owner Tracking**: Track page allocation sources

## Performance Optimizations

### Fast Path Optimizations
- **Branch Prediction**: Optimize common case branches
- **Cache Line Optimization**: Minimize cache line usage
- **Lock Optimization**: Reduce lock contention
- **Inline Functions**: Inline critical path functions

### Memory Access Patterns
- **Prefetching**: Prefetch likely-to-be-accessed data
- **Cache Locality**: Maintain spatial and temporal locality
- **NUMA Optimization**: Minimize cross-node access
- **TLB Efficiency**: Optimize TLB usage patterns

### Batch Operations
```c
unsigned long alloc_pages_bulk_noprof(gfp_t gfp, int preferred_nid,
                                     nodemask_t *nodemask, int nr_pages,
                                     struct list_head *page_list,
                                     struct page **page_array)
```

**Bulk Allocation Benefits**:
- **Amortized Overhead**: Reduce per-page overhead
- **Lock Efficiency**: Minimize lock acquisition
- **Cache Optimization**: Better cache utilization
- **Batch Processing**: Optimize for bulk consumers

## Integration Points

### Memory Management Subsystems
- **Slab Allocator**: Provides pages for slab allocation
- **Virtual Memory**: Handles page faults and demand paging
- **Swap System**: Provides pages for swap cache
- **File System Cache**: Allocates pages for page cache

### Hardware Integration
- **DMA Constraints**: Handle DMA memory requirements
- **NUMA Topology**: Adapt to hardware NUMA layout
- **Memory Controllers**: Work with hardware memory controllers
- **Cache Hierarchy**: Consider CPU cache organization

### Control Groups (cgroups)
- **Memory Limits**: Enforce per-cgroup memory limits
- **Accounting**: Track memory usage per cgroup
- **Reclaim Integration**: Coordinate with cgroup reclaim
- **OOM Handling**: Handle per-cgroup OOM conditions

This implementation represents the heart of kernel memory management, providing efficient, scalable, and secure page allocation services that support all other kernel subsystems while handling the complexities of modern hardware architectures, NUMA systems, and memory hotplug scenarios.