# Linux Kernel Page Allocator Implementation (mm/page_alloc.c)

## Table of Contents
1. [Overview and Architecture](#overview)
2. [Buddy Allocator Algorithm and Free Area Management](#buddy-allocator)
3. [Per-CPU Page Lists and Performance Optimizations](#per-cpu-lists)
4. [Memory Zone Management and NUMA Architecture](#zone-management)
5. [Memory Pressure Handling and Reclaim Integration](#memory-pressure)
6. [Debug Infrastructure and Memory Tracking](#debug-infrastructure)
7. [API Design and Allocation Strategies](#api-design)
8. [Integration with Memory Management Subsystems](#integration)

## Overview and Architecture {#overview}

The Linux kernel page allocator (`mm/page_alloc.c`) is the central component of the memory management subsystem responsible for managing physical page allocation and deallocation. The allocator implements a zoned buddy system that efficiently manages free memory across different memory zones while maintaining optimal performance through various caching and optimization strategies.

### Key Design Principles

- **Zoned Architecture**: Memory is divided into zones (DMA, Normal, HighMem) with specific properties
- **Buddy System**: Binary buddy algorithm for efficient coalescing and fragmentation management
- **Per-CPU Caching**: Hot/cold page lists for improved performance and NUMA locality
- **Migration Types**: Pages grouped by mobility to reduce fragmentation
- **Watermark-based Management**: Proactive memory pressure detection and handling

### Core Data Structures

```c
// Free area management for buddy allocator
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];  // Per-migration-type freelists
    unsigned long nr_free;                      // Number of free pages
};

// Per-CPU page cache for performance optimization
struct per_cpu_pages {
    spinlock_t lock;        // Protects lists field
    int count;              // Number of pages in the list
    int high;               // High watermark, emptying needed
    int high_min;           // Min high watermark
    int high_max;           // Max high watermark
    int batch;              // Chunk size for buddy add/remove
    u8 flags;               // Protected by pcp->lock
    u8 alloc_factor;        // Batch scaling factor during allocate
    #ifdef CONFIG_NUMA
    u8 expire;              // When 0, remote pagesets are drained
    #endif
    short free_count;       // Consecutive free count
    struct list_head lists[NR_PCP_LISTS];  // Lists per migrate type
};
```

## Buddy Allocator Algorithm and Free Area Management {#buddy-allocator}

### Algorithm Overview

The buddy allocator is the core algorithm for managing physical memory pages. It maintains free pages in power-of-2 sized blocks, allowing efficient allocation and coalescing of contiguous memory regions.

### Key Features

1. **Binary Tree Structure**: Pages organized in orders 0 to MAX_PAGE_ORDER (typically 10)
2. **Coalescing**: Adjacent free blocks automatically merge to form larger blocks
3. **Splitting**: Large blocks split into smaller ones when needed
4. **Migration Type Awareness**: Separate freelists for different page mobility types

### Core Functions

#### __free_one_page() - The Heart of Buddy Deallocation

```c
static inline void __free_one_page(struct page *page,
        unsigned long pfn,
        struct zone *zone, unsigned int order,
        int migratetype, fpi_t fpi_flags)
{
    struct capture_control *capc = task_capc(zone);
    unsigned long buddy_pfn = 0;
    unsigned long combined_pfn;
    struct page *buddy;
    bool to_tail;

    // Validate inputs and zone state
    VM_BUG_ON(!zone_is_initialized(zone));
    VM_BUG_ON_PAGE(page->flags & PAGE_FLAGS_CHECK_AT_PREP, page);
    VM_BUG_ON(migratetype == -1);
    VM_BUG_ON_PAGE(pfn & ((1 << order) - 1), page);
    VM_BUG_ON_PAGE(bad_range(zone, page), page);

    account_freepages(zone, 1 << order, migratetype);

    // Buddy coalescing loop
    while (order < MAX_PAGE_ORDER) {
        int buddy_mt = migratetype;

        // Check for memory compaction capture
        if (compaction_capture(capc, page, order, migratetype)) {
            account_freepages(zone, -(1 << order), migratetype);
            return;
        }

        // Find buddy page
        buddy = find_buddy_page_pfn(page, pfn, order, &buddy_pfn);
        if (!buddy)
            goto done_merging;

        // Handle pageblock-level migration type differences
        if (unlikely(order >= pageblock_order)) {
            buddy_mt = get_pfnblock_migratetype(buddy, buddy_pfn);
            if (migratetype != buddy_mt &&
                (!migratetype_is_mergeable(migratetype) ||
                 !migratetype_is_mergeable(buddy_mt)))
                goto done_merging;
        }

        // Merge with buddy
        if (page_is_guard(buddy))
            clear_page_guard(zone, buddy, order);
        else
            __del_page_from_free_list(buddy, zone, order, buddy_mt);

        // Ensure consistent migration type
        if (unlikely(buddy_mt != migratetype))
            set_pageblock_migratetype(buddy, migratetype);

        // Move to next level
        combined_pfn = buddy_pfn & pfn;
        page = page + (combined_pfn - pfn);
        pfn = combined_pfn;
        order++;
    }

done_merging:
    set_buddy_order(page, order);

    // Determine insertion position (head/tail)
    if (fpi_flags & FPI_TO_TAIL)
        to_tail = true;
    else if (is_shuffle_order(order))
        to_tail = shuffle_pick_tail();
    else
        to_tail = buddy_merge_likely(pfn, buddy_pfn, page, order);

    __add_to_free_list(page, zone, order, migratetype, to_tail);

    // Notify page reporting subsystem
    if (!(fpi_flags & FPI_SKIP_REPORT_NOTIFY))
        page_reporting_notify_free(order);
}
```

#### rmqueue_buddy() - Buddy Allocation Interface

```c
struct page *rmqueue_buddy(struct zone *preferred_zone, struct zone *zone,
               unsigned int order, unsigned int alloc_flags,
               int migratetype)
{
    struct page *page;
    unsigned long flags;

    do {
        page = NULL;
        
        // Handle trylock vs regular locking
        if (unlikely(alloc_flags & ALLOC_TRYLOCK)) {
            if (!spin_trylock_irqsave(&zone->lock, flags))
                return NULL;
        } else {
            spin_lock_irqsave(&zone->lock, flags);
        }

        // Try high atomic reserves first if needed
        if (alloc_flags & ALLOC_HIGHATOMIC)
            page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
        
        if (!page) {
            enum rmqueue_mode rmqm = RMQUEUE_NORMAL;
            page = __rmqueue(zone, order, migratetype, alloc_flags, &rmqm);

            // Fallback to high atomic for critical allocations
            if (!page && (alloc_flags & (ALLOC_OOM|ALLOC_NON_BLOCK)))
                page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);

            if (!page) {
                spin_unlock_irqrestore(&zone->lock, flags);
                return NULL;
            }
        }
        spin_unlock_irqrestore(&zone->lock, flags);
    } while (check_new_pages(page, order));

    // Update statistics
    __count_zid_vm_events(PGALLOC, page_zonenum(page), 1 << order);
    zone_statistics(preferred_zone, zone, 1);

    return page;
}
```

### Migration Types and Fragmentation Management

The allocator groups pages by mobility to reduce long-term fragmentation:

```c
enum migratetype {
    MIGRATE_UNMOVABLE,      // Cannot be moved (kernel allocations)
    MIGRATE_MOVABLE,        // Can be moved (user pages, page cache)
    MIGRATE_RECLAIMABLE,    // Can be reclaimed (slab cache)
    MIGRATE_PCPTYPES,       // Number of types on PCP lists
    MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,  // High priority atomic
    #ifdef CONFIG_CMA
    MIGRATE_CMA,            // Contiguous Memory Allocator
    #endif
    #ifdef CONFIG_MEMORY_ISOLATION
    MIGRATE_ISOLATE,        // Isolated pages
    #endif
    MIGRATE_TYPES
};
```

## Per-CPU Page Lists and Performance Optimizations {#per-cpu-lists}

### Overview

Per-CPU page lists (PCPs) are a critical performance optimization that reduces lock contention and improves cache locality by maintaining per-CPU caches of recently freed pages.

### PCP Architecture

#### Structure and Configuration

```c
struct per_cpu_pages {
    spinlock_t lock;        // Protects the entire structure
    int count;              // Current number of pages cached
    int high;               // High watermark for cache emptying
    int high_min;           // Minimum high watermark
    int high_max;           // Maximum high watermark
    int batch;              // Number of pages to add/remove in batch
    u8 flags;               // Various flags
    u8 alloc_factor;        // Scaling factor for allocation batches
    #ifdef CONFIG_NUMA
    u8 expire;              // Expiration counter for remote drainage
    #endif
    short free_count;       // Count of consecutive frees
    struct list_head lists[NR_PCP_LISTS];  // Per-migration-type lists
};
```

#### Batch Allocation Logic

The PCP system uses sophisticated batching to optimize performance:

```c
static int nr_pcp_alloc(struct per_cpu_pages *pcp, struct zone *zone, int order)
{
    int high, base_batch, batch, max_nr_alloc;
    int high_max, high_min;

    base_batch = READ_ONCE(pcp->batch);
    high_min = READ_ONCE(pcp->high_min);
    high_max = READ_ONCE(pcp->high_max);
    high = pcp->high = clamp(pcp->high, high_min, high_max);

    // Check for disabled PCP or boot pageset
    if (unlikely(high < base_batch))
        return 1;

    if (order)
        batch = base_batch;                          // Higher orders use base
    else
        batch = (base_batch << pcp->alloc_factor);   // Scale for order-0

    // Dynamic high watermark adjustment
    if (high_min != high_max && !test_bit(ZONE_BELOW_HIGH, &zone->flags))
        high = pcp->high = min(high + batch, high_max);

    if (!order) {
        max_nr_alloc = max(high - pcp->count - base_batch, base_batch);
        max_nr_alloc = max(batch, max_nr_alloc);
        max_nr_alloc = min(max_nr_alloc, high - pcp->count);
    } else {
        max_nr_alloc = batch;
    }

    return max_nr_alloc;
}
```

### PCP Locking and Concurrency

The PCP system uses sophisticated locking mechanisms to balance performance and correctness:

```c
// Task pinning for PCP access
#ifndef CONFIG_PREEMPT_RT
#define pcpu_task_pin()     preempt_disable()
#define pcpu_task_unpin()   preempt_enable()
#else
#define pcpu_task_pin()     migrate_disable()
#define pcpu_task_unpin()   migrate_enable()
#endif

// Generic per-CPU spinlock helper
#define pcpu_spin_lock(type, member, ptr) ({            \
    type *_ret;                                         \
    pcpu_task_pin();                                    \
    _ret = this_cpu_ptr(ptr);                          \
    spin_lock(&_ret->member);                          \
    _ret;                                              \
})

// PCP-specific helpers
#define pcp_spin_lock(ptr)                              \
    pcpu_spin_lock(struct per_cpu_pages, lock, ptr)

#define pcp_spin_unlock(ptr)                            \
    pcpu_spin_unlock(lock, ptr)
```

### Hot/Cold Page Management

The PCP system maintains both hot (recently allocated) and cold (least recently used) pages to optimize cache behavior:

1. **Hot Pages**: Recently freed pages likely still in CPU cache
2. **Cold Pages**: Pages that have been in the PCP for longer
3. **Allocation Strategy**: Prefer hot pages for better cache locality
4. **Deallocation Strategy**: Add to hot end, expire from cold end

### NUMA Considerations

For NUMA systems, the PCP implementation includes special handling:

```c
#ifdef CONFIG_NUMA
u8 expire;              // Expiration counter for remote node pages
#endif

// NUMA node handling
#ifdef CONFIG_USE_PERCPU_NUMA_NODE_ID
DEFINE_PER_CPU(int, numa_node);
EXPORT_PER_CPU_SYMBOL(numa_node);
#endif

#ifdef CONFIG_HAVE_MEMORYLESS_NODES
DEFINE_PER_CPU(int, _numa_mem_);    // Kernel "local memory" node
EXPORT_PER_CPU_SYMBOL(_numa_mem_);
#endif
```

## Memory Zone Management and NUMA Architecture {#zone-management}

### Zone Types and Properties

The Linux kernel divides physical memory into zones based on hardware constraints and usage patterns:

```c
enum zone_type {
    #ifdef CONFIG_ZONE_DMA
    ZONE_DMA,              // DMA-capable memory (< 16MB on x86)
    #endif
    #ifdef CONFIG_ZONE_DMA32
    ZONE_DMA32,            // DMA32-capable memory (< 4GB on x86_64)
    #endif
    ZONE_NORMAL,           // Normal memory
    #ifdef CONFIG_HIGHMEM
    ZONE_HIGHMEM,          // High memory (> 896MB on 32-bit x86)
    #endif
    ZONE_MOVABLE,          // Movable memory for memory hotplug
    __MAX_NR_ZONES
};
```

### Zone Structure and Management

Each memory zone maintains its own set of management structures:

```c
struct zone {
    // Watermarks for memory pressure detection
    unsigned long _watermark[NR_WMARK];     // MIN, LOW, HIGH watermarks
    unsigned long watermark_boost;          // Boost for fragmentation

    // Free page management
    struct free_area free_area[MAX_PAGE_ORDER + 1];  // Buddy allocator
    
    // Per-CPU page management
    struct per_cpu_pages __percpu *per_cpu_pageset;
    struct per_cpu_zonestat __percpu *per_cpu_zonestats;
    
    // Zone configuration
    int pageset_high_min;
    int pageset_high_max;
    int pageset_batch;
    
    // Zone properties
    unsigned long zone_start_pfn;           // First PFN in zone
    unsigned long managed_pages;            // Pages managed by buddy
    unsigned long spanned_pages;            // Total pages in zone
    unsigned long present_pages;            // Physical pages present
    
    // NUMA node information
    struct pglist_data *zone_pgdat;         // Parent NUMA node
    
    // Locking
    spinlock_t lock;                        // Zone lock
    
    // Zone state flags
    unsigned long flags;                    // Zone status flags
};
```

### Zone Selection and Fallback

The allocator implements a sophisticated zone selection mechanism with fallback order: MOVABLE => HIGHMEM => NORMAL => DMA32 => DMA.

#### get_page_from_freelist() - Zone Traversal

```c
static struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
                      const struct alloc_context *ac)
{
    struct zoneref *z;
    struct zone *zone;
    struct pglist_data *last_pgdat = NULL;
    bool last_pgdat_dirty_ok = false;
    bool no_fallback;

retry:
    no_fallback = alloc_flags & ALLOC_NOFRAGMENT;
    z = ac->preferred_zoneref;
    
    // Traverse zones in order of preference
    for_next_zone_zonelist_nodemask(zone, z, ac->highest_zoneidx,
                    ac->nodemask) {
        struct page *page;
        unsigned long mark;

        // Check cpuset constraints
        if (cpusets_enabled() &&
            (alloc_flags & ALLOC_CPUSET) &&
            !__cpuset_zone_allowed(zone, gfp_mask))
                continue;

        // Check dirty limits for writeback throttling
        if (ac->spread_dirty_pages) {
            if (last_pgdat != zone->zone_pgdat) {
                last_pgdat = zone->zone_pgdat;
                last_pgdat_dirty_ok = node_dirty_ok(zone->zone_pgdat);
            }
            if (!last_pgdat_dirty_ok)
                continue;
        }

        // NUMA locality check
        if (no_fallback && !defrag_mode && nr_online_nodes > 1 &&
            zone != zonelist_zone(ac->preferred_zoneref)) {
            int local_nid = zonelist_node_idx(ac->preferred_zoneref);
            if (zone_to_nid(zone) != local_nid) {
                alloc_flags &= ~ALLOC_NOFRAGMENT;
                goto retry;
            }
        }

        // Memory acceptance for confidential computing
        cond_accept_memory(zone, order, alloc_flags);

        // Watermark checking
        if (test_bit(ZONE_BELOW_HIGH, &zone->flags))
            goto check_alloc_wmark;

        mark = high_wmark_pages(zone);
        if (zone_watermark_fast(zone, order, mark,
                    ac->highest_zoneidx, alloc_flags, gfp_mask))
            goto try_this_zone;
        else
            set_bit(ZONE_BELOW_HIGH, &zone->flags);

check_alloc_wmark:
        mark = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK);
        if (!zone_watermark_fast(zone, order, mark,
                       ac->highest_zoneidx, alloc_flags, gfp_mask)) {
            // Handle failed allocation attempts
            // ... (continue with reclaim, compaction, etc.)
        }

try_this_zone:
        page = rmqueue(ac->preferred_zoneref->zone, zone, order,
                gfp_mask, alloc_flags, ac->migratetype);
        if (page) {
            prep_new_page(page, order, gfp_mask, alloc_flags);
            return page;
        }
    }

    // No suitable zone found
    return NULL;
}
```

### NUMA Node Management

#### Node Distance and Allocation Policy

```c
// NUMA statistics tracking
#ifdef CONFIG_NUMA
enum numa_stat_item {
    NUMA_HIT,              // Allocated in intended node
    NUMA_MISS,             // Allocated in non-intended node  
    NUMA_FOREIGN,          // Was intended here, hit elsewhere
    NUMA_INTERLEAVE_HIT,   // Interleaver preferred this zone
    NUMA_LOCAL,            // Allocation from local node
    NUMA_OTHER,            // Allocation from other node
    NR_VM_NUMA_EVENT_ITEMS
};

// NUMA statistics updating
static inline void zone_statistics(struct zone *preferred_zone, struct zone *z,
                   long nr_account)
{
#ifdef CONFIG_NUMA
    enum numa_stat_item local_stat = NUMA_LOCAL;

    if (zone_to_nid(z) != zone_to_nid(preferred_zone)) {
        local_stat = NUMA_OTHER;
        __count_numa_events(preferred_zone, NUMA_FOREIGN, nr_account);
    }
    __count_numa_events(z, local_stat, nr_account);
#endif
}
#endif
```

## Memory Pressure Handling and Reclaim Integration {#memory-pressure}

### Watermark-Based Management

The page allocator uses a three-level watermark system to detect and respond to memory pressure:

```c
enum wmark_pages {
    WMARK_MIN,              // Minimum free pages - direct reclaim threshold
    WMARK_LOW,              // Low free pages - kswapd wakeup threshold  
    WMARK_HIGH,             // High free pages - kswapd sleep threshold
    NR_WMARK
};
```

#### Watermark Checking Implementation

```c
bool __zone_watermark_ok(struct zone *z, unsigned int order, unsigned long mark,
            int highest_zoneidx, unsigned int alloc_flags,
            long free_pages)
{
    long min = mark;
    int reserve = 0;
    int alloc_harder = 0;
    bool atomic = alloc_flags & ALLOC_ATOMIC;

    // Account for unusable free pages
    free_pages -= __zone_watermark_unusable_free(z, order, alloc_flags);

    // High atomic reserves
    if (unlikely(alloc_flags & ALLOC_HIGHATOMIC))
        reserve -= z->nr_reserved_highatomic;

    // Reserve adjustments for different allocation contexts
    if (alloc_flags & ALLOC_HARDER) {
        alloc_harder = ALLOC_HARDER;
        reserve -= min / 4;
    }

    if (atomic) {
        if (!(alloc_flags & ALLOC_CPUSET) || reserve) {
            alloc_harder = ALLOC_HARDER;
            reserve -= min / 8;
        }
    }

    // Check against adjusted watermark
    if (free_pages <= min + reserve)
        return false;

    // For higher order allocations, check free area availability
    if (order) {
        // Complex logic to check if sufficient large blocks exist
        // ... (detailed implementation for anti-fragmentation)
    }

    return true;
}

// Fast watermark check with optimizations
static inline bool zone_watermark_fast(struct zone *z, unsigned int order,
        unsigned long mark, int highest_zoneidx,
        unsigned int alloc_flags, gfp_t gfp_mask)
{
    long free_pages;

    free_pages = zone_page_state(z, NR_FREE_PAGES);

    // Quick check for obviously sufficient memory
    if (free_pages > mark + z->lowmem_reserve[highest_zoneidx])
        return true;

    // Detailed check for marginal cases
    return __zone_watermark_ok(z, order, mark, highest_zoneidx, alloc_flags,
                  free_pages);
}
```

### kswapd Integration

The page allocator integrates closely with kswapd (kernel swap daemon) for proactive memory reclaim.

### Direct Reclaim Integration

When kswapd cannot keep up with memory pressure, the allocator triggers direct reclaim:

```c
static inline struct page *
__alloc_pages_direct_reclaim(gfp_t gfp_mask, unsigned int order,
        unsigned int alloc_flags, const struct alloc_context *ac,
        unsigned long *did_some_progress)
{
    struct page *page = NULL;
    unsigned long pflags;
    bool drained = false;

    psi_memstall_enter(&pflags);
    *did_some_progress = __perform_reclaim(gfp_mask, order, ac);
    if (unlikely(!(*did_some_progress)))
        goto out;

retry:
    page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);

    // If allocation still fails, try draining per-CPU lists
    if (!page && !drained) {
        unreserve_highatomic_pageblock(ac, false);
        drain_all_pages(NULL);
        drained = true;
        goto retry;
    }

out:
    psi_memstall_leave(&pflags);
    return page;
}
```

### OOM (Out of Memory) Handling

When all reclaim attempts fail, the allocator may invoke the OOM killer for critical allocations that cannot fail.

## Debug Infrastructure and Memory Tracking {#debug-infrastructure}

### Page Allocation Debugging

The kernel provides comprehensive debugging facilities for page allocation:

#### Debug Page Allocation

```c
#ifdef CONFIG_DEBUG_PAGEALLOC
// Unmap pages from kernel direct mapping to detect use-after-free
static inline void debug_pagealloc_unmap_pages(struct page *page, int numpages)
{
    if (!debug_pagealloc_enabled())
        return;
        
    kernel_map_pages(page, numpages, 0);
}

// Map pages back when allocating
static inline void debug_pagealloc_map_pages(struct page *page, int numpages)
{
    if (!debug_pagealloc_enabled())
        return;
        
    kernel_map_pages(page, numpages, 1);
}
#endif
```

#### Page Owner Tracking

```c
#ifdef CONFIG_PAGE_OWNER
// Capture allocation stack trace
void __set_page_owner(struct page *page, unsigned int order,
              gfp_t gfp_mask, short last_migrate_reason)
{
    struct page_owner *page_owner;
    depot_stack_handle_t handle;

    handle = save_stack(GFP_NOWAIT | __GFP_NOWARN);
    
    page_owner = get_page_owner(page_to_pfn(page));
    page_owner->handle = handle;
    page_owner->order = order;
    page_owner->gfp_mask = gfp_mask;
    page_owner->last_migrate_reason = last_migrate_reason;
    page_owner->pid = current->pid;
    page_owner->tgid = current->tgid;
    page_owner->ts_nsec = local_clock();
    strlcpy(page_owner->comm, current->comm, sizeof(page_owner->comm));
    
    __set_bit(PAGE_OWNER_ALLOCATED, &page->page_owner_ops);
}
#endif
```

### Memory Leak Detection

#### KMEMLEAK Integration

```c
#ifdef CONFIG_DEBUG_KMEMLEAK
static inline void kmemleak_alloc_page(struct page *page, unsigned int order,
                       gfp_t gfp)
{
    kmemleak_alloc(__va(page_to_pfn(page) << PAGE_SHIFT),
               PAGE_SIZE << order, 1, gfp);
}

static inline void kmemleak_free_page(struct page *page)
{
    kmemleak_free(__va(page_to_pfn(page) << PAGE_SHIFT));
}
#endif
```

### Page Poisoning

```c
#ifdef CONFIG_PAGE_POISONING
// Fill freed pages with poison pattern
static void poison_pages(struct page *page, int n, unsigned char val)
{
    int i;
    
    for (i = 0; i < n; i++) {
        void *addr = kmap_atomic(page + i);
        memset(addr, val, PAGE_SIZE);
        kunmap_atomic(addr);
    }
}
#endif
```

### Statistics and Monitoring

#### VM Statistics

```c
// Zone-specific statistics
enum zone_stat_item {
    NR_FREE_PAGES,
    NR_ZONE_LRU_BASE,      // Start of LRU lists
    NR_ZONE_INACTIVE_ANON = NR_ZONE_LRU_BASE,
    NR_ZONE_ACTIVE_ANON,
    NR_ZONE_INACTIVE_FILE,
    NR_ZONE_ACTIVE_FILE,
    NR_ZONE_UNEVICTABLE,
    NR_ZONE_WRITE_PENDING, // Pages under writeback
    NR_MLOCK,              // mlocked pages
    NR_PAGETABLE,          // Page table pages
    NR_BOUNCE,             // Bounce buffer pages
    NR_VM_ZONE_STAT_ITEMS
};

// Global VM statistics
enum vm_event_item {
    PGALLOC_DMA,
    PGALLOC_DMA32,
    PGALLOC_NORMAL,
    PGALLOC_MOVABLE,
    PGFREE,
    PGACTIVATE,
    PGDEACTIVATE,
    PGSCAN_KSWAPD,
    PGSCAN_DIRECT,
    PGSTEAL_KSWAPD,
    PGSTEAL_DIRECT,
    // ... many more events
    NR_VM_EVENT_ITEMS
};
```

## API Design and Allocation Strategies {#api-design}

### Primary Allocation APIs

#### Core Allocation Functions

```c
// Main allocation entry point
struct page *__alloc_pages_noprof(gfp_t gfp, unsigned int order,
                  int preferred_nid, nodemask_t *nodemask)
{
    struct page *page;
    unsigned int alloc_flags = ALLOC_WMARK_LOW;
    gfp_t alloc_gfp; 
    struct alloc_context ac = { };

    // Input validation and setup
    if (unlikely(order >= MAX_PAGE_ORDER)) {
        WARN_ON_ONCE(!(gfp & __GFP_NOWARN));
        return NULL;
    }

    gfp &= gfp_allowed_mask;
    alloc_gfp = gfp;
    if (!prepare_alloc_pages(gfp, order, preferred_nid, nodemask, &ac,
            &alloc_gfp, &alloc_flags))
        return NULL;

    // Set up allocation context
    ac.highest_zoneidx = gfp_zone(gfp);
    ac.zonelist = node_zonelist(preferred_nid, gfp);
    ac.nodemask = nodemask;
    ac.migratetype = gfp_migratetype(gfp);

    // Add fragmentation avoidance flag if appropriate
    alloc_flags |= alloc_flags_nofragment(zonelist_zone(ac.preferred_zoneref), gfp);

    // First allocation attempt
    page = get_page_from_freelist(alloc_gfp, order, alloc_flags, &ac);
    if (likely(page))
        goto out;

    // Slow path if fast allocation failed
    alloc_gfp = gfp;
    ac.spread_dirty_pages = false;

    page = __alloc_pages_slowpath(alloc_gfp, order, &ac);

out:
    if (memcg_kmem_online() && (gfp & __GFP_ACCOUNT) && page &&
        unlikely(__memcg_kmem_charge_page(page, gfp, order) != 0)) {
        __free_pages(page, order);
        page = NULL;
    }

    trace_mm_page_alloc(page, order, alloc_gfp, ac.migratetype);
    return page;
}

// Wrapper with memory allocation profiling
#define alloc_pages(gfp_mask, order) \
    alloc_hooks(__alloc_pages_noprof(gfp_mask, order, numa_node_id(), NULL))
```

#### Bulk Allocation API

The kernel provides efficient bulk allocation for scenarios requiring multiple pages:

```c
// Efficient bulk allocation for multiple pages
unsigned long __alloc_pages_bulk(gfp_t gfp, int preferred_nid,
                nodemask_t *nodemask, int nr_pages,
                struct list_head *page_list,
                struct page **page_array)
```

### GFP Flag System

The GFP (Get Free Pages) flag system provides fine-grained control over allocation behavior:

#### Core GFP Flags

```c
// Reclaim flags
#define __GFP_DIRECT_RECLAIM    ((__force gfp_t)___GFP_DIRECT_RECLAIM)
#define __GFP_KSWAPD_RECLAIM    ((__force gfp_t)___GFP_KSWAPD_RECLAIM)
#define __GFP_RECLAIM           ((__force gfp_t)(___GFP_DIRECT_RECLAIM|___GFP_KSWAPD_RECLAIM))

// Zone modifiers
#define __GFP_DMA               ((__force gfp_t)___GFP_DMA)
#define __GFP_HIGHMEM           ((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_DMA32             ((__force gfp_t)___GFP_DMA32)
#define __GFP_MOVABLE           ((__force gfp_t)___GFP_MOVABLE)

// Mobility and placement flags
#define __GFP_RECLAIMABLE       ((__force gfp_t)___GFP_RECLAIMABLE)
#define __GFP_WRITE             ((__force gfp_t)___GFP_WRITE)
#define __GFP_HARDWALL          ((__force gfp_t)___GFP_HARDWALL)
#define __GFP_THISNODE          ((__force gfp_t)___GFP_THISNODE)

// Common flag combinations
#define GFP_ATOMIC              (__GFP_HIGH|__GFP_ATOMIC|__GFP_KSWAPD_RECLAIM)
#define GFP_KERNEL              (__GFP_RECLAIM | __GFP_IO | __GFP_FS)
#define GFP_KERNEL_ACCOUNT      (GFP_KERNEL | __GFP_ACCOUNT)
#define GFP_NOWAIT              (__GFP_KSWAPD_RECLAIM)
#define GFP_NOIO                (__GFP_RECLAIM)
#define GFP_NOFS                (__GFP_RECLAIM | __GFP_IO)
#define GFP_USER                (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
#define GFP_DMA                 (__GFP_DMA)
#define GFP_DMA32               (__GFP_DMA32)
#define GFP_HIGHUSER            (GFP_USER | __GFP_HIGHMEM)
#define GFP_HIGHUSER_MOVABLE    (GFP_HIGHUSER | __GFP_MOVABLE)
```

#### Migration Type Mapping

```c
static inline int gfp_migratetype(const gfp_t gfp_flags)
{
    VM_WARN_ON((gfp_flags & GFP_MOVABLE_MASK) == GFP_MOVABLE_MASK);
    BUILD_BUG_ON((1UL << GFP_MOVABLE_SHIFT) != ___GFP_MOVABLE);
    BUILD_BUG_ON((___GFP_MOVABLE >> GFP_MOVABLE_SHIFT) != MIGRATE_MOVABLE);
    BUILD_BUG_ON((___GFP_RECLAIMABLE >> GFP_MOVABLE_SHIFT) != MIGRATE_RECLAIMABLE);
    BUILD_BUG_ON(((___GFP_MOVABLE | ___GFP_RECLAIMABLE) >>
              GFP_MOVABLE_SHIFT) != MIGRATE_HIGHATOMIC);

    if (unlikely(page_group_by_mobility_disabled))
        return MIGRATE_UNMOVABLE;

    // Group based on mobility
    return (__force unsigned long)(gfp_flags & GFP_MOVABLE_MASK) >> GFP_MOVABLE_SHIFT;
}
```

### Allocation Context and Strategy

#### Allocation Context Structure

```c
struct alloc_context {
    struct zonelist *zonelist;      // Ordered list of zones to try
    nodemask_t *nodemask;           // Allowed NUMA nodes
    struct zoneref *preferred_zoneref;  // Preferred zone reference
    int migratetype;                // Migration type for this allocation
    enum zone_type highest_zoneidx; // Highest zone index allowed
    bool spread_dirty_pages;        // Spread dirty pages across nodes
};
```

## Integration with Memory Management Subsystems {#integration}

### Memory Compaction Integration

The page allocator works closely with memory compaction to reduce fragmentation:

```c
// Compaction integration in allocation path
static struct page *
__alloc_pages_direct_compact(gfp_t gfp_mask, unsigned int order,
        unsigned int alloc_flags, const struct alloc_context *ac,
        enum compact_priority priority, enum compact_result *compact_result)
{
    struct page *page = NULL;
    unsigned long pflags;
    unsigned int noreclaim_flag;

    if (!order)
        return NULL;

    psi_memstall_enter(&pflags);
    delayacct_compact_start();
    noreclaim_flag = memalloc_noreclaim_save();

    *compact_result = try_to_compact_pages(gfp_mask, order, alloc_flags, ac,
                          priority, NULL);

    memalloc_noreclaim_restore(noreclaim_flag);
    psi_memstall_leave(&pflags);
    delayacct_compact_end();

    if (*compact_result == COMPACT_SKIPPED)
        return NULL;

    page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);

    if (page) {
        struct zone *zone = page_zone(page);

        zone->compact_blockskip_flush = false;
        compaction_defer_reset(zone, order, true);
        count_vm_event(COMPACTSUCCESS);
        return page;
    }

    // Compaction failed, adjust priority and retry limits
    count_vm_event(COMPACTFAIL);
    cond_resched();

    return NULL;
}
```

### Memory Hotplug Integration

```c
// Memory hotplug support
void zone_pcp_update(struct zone *zone, int cpu_online)
{
    mutex_lock(&pcp_batch_high_lock);
    zone_pcp_update_cacheinfo(zone, cpu_online);
    mutex_unlock(&pcp_batch_high_lock);
}

// Online memory callback
static int page_alloc_cpu_online(unsigned int cpu)
{
    struct zone *zone;

    for_each_populated_zone(zone) {
        zone_pcp_update(zone, 1);
    }
    return 0;
}

// Offline memory callback  
static int page_alloc_cpu_dead(unsigned int cpu)
{
    struct zone *zone;

    for_each_populated_zone(zone) {
        zone_pcp_update(zone, 0);
    }
    
    drain_pages(cpu);
    return 0;
}
```

### CMA (Contiguous Memory Allocator) Integration

```c
#ifdef CONFIG_CMA
// CMA integration for contiguous allocations
bool is_migrate_cma(int migratetype)
{
    return migratetype == MIGRATE_CMA;
}

// CMA page allocation
struct page *alloc_contig_pages(unsigned long nr_pages, gfp_t gfp_mask,
                int nid, nodemask_t *nodemask)
{
    unsigned long ret, pfn, flags;
    struct zonelist *zonelist;
    struct zone *zone;
    struct zoneref *z;

    zonelist = node_zonelist(nid, gfp_mask);
    for_each_zone_zonelist_nodemask(zone, z, zonelist,
                    gfp_zone(gfp_mask), nodemask) {
        spin_lock_irqsave(&zone->lock, flags);

        pfn = ALIGN(zone->zone_start_pfn, nr_pages);
        while (zone_spans_pfn(zone, pfn + nr_pages - 1)) {
            ret = alloc_contig_range(pfn, pfn + nr_pages,
                        MIGRATE_MOVABLE, gfp_mask);
            if (!ret)
                return pfn_to_page(pfn);
            pfn += MAX_ORDER_NR_PAGES;
        }
        spin_unlock_irqrestore(&zone->lock, flags);
    }
    return NULL;
}
#endif
```

### Memory Control Group Integration

```c
#ifdef CONFIG_MEMCG
// Memory cgroup integration for accounting
static inline int __memcg_kmem_charge_page(struct page *page, gfp_t gfp,
                       int order)
{
    struct obj_cgroup *objcg;
    int ret = 0;

    objcg = get_obj_cgroup_from_current();
    if (objcg) {
        ret = obj_cgroup_charge_pages(objcg, gfp, 1 << order);
        if (!ret) {
            page->memcg_data = (unsigned long)objcg |
                      MEMCG_DATA_KMEM;
            return 0;
        }
        obj_cgroup_put(objcg);
    }
    return ret;
}

static inline void __memcg_kmem_uncharge_page(struct page *page, int order)
{
    struct obj_cgroup *objcg;
    unsigned int nr_pages = 1 << order;

    if (!PageMemcgKmem(page))
        return;

    objcg = __page_objcg(page);
    obj_cgroup_uncharge_pages(objcg, nr_pages);
    page->memcg_data = 0;
    obj_cgroup_put(objcg);
}
#endif
```

### Swap and Reclaim Integration

```c
// Watermark boost for memory pressure
static void boost_watermark(struct zone *zone)
{
    unsigned long max_boost;

    if (!watermark_boost_factor)
        return;

    // Boost up to 1.5% of the high watermark
    max_boost = mult_frac(zone->_watermark[WMARK_HIGH],
                watermark_boost_factor, 10000);

    // Ensure boost doesn't make high > max possible pages
    max_boost = min(max_boost, zone->managed_pages / 2);

    zone->watermark_boost = min(zone->watermark_boost + pageblock_nr_pages,
                   max_boost);
}

// Kswapd wakeup integration
void wakeup_kswapd(struct zone *zone, gfp_t gfp_flags, int order,
           enum zone_type highest_zoneidx)
{
    pg_data_t *pgdat;
    enum zone_type curr_idx;

    if (!managed_zone(zone))
        return;

    if (!cpuset_zone_allowed(zone, gfp_flags))
        return;

    pgdat = zone->zone_pgdat;
    curr_idx = READ_ONCE(pgdat->kswapd_highest_zoneidx);

    if (curr_idx == MAX_NR_ZONES || curr_idx < highest_zoneidx)
        WRITE_ONCE(pgdat->kswapd_highest_zoneidx, highest_zoneidx);

    if (!waitqueue_active(&pgdat->kswapd_wait))
        return;

    // Only wake kswapd if it's asleep or if it's reclaiming for lower zones
    if (pgdat->kswapd_order < order ||
        pgdat->kswapd_highest_zoneidx < highest_zoneidx) {
        WRITE_ONCE(pgdat->kswapd_order, order);
        WRITE_ONCE(pgdat->kswapd_highest_zoneidx, highest_zoneidx);
        wake_up_interruptible(&pgdat->kswapd_wait);
    }
}
```

## Conclusion

The Linux kernel page allocator is a sophisticated system that balances performance, memory efficiency, and system stability. Key aspects include:

1. **Scalable Architecture**: The buddy allocator with per-CPU caching provides excellent scalability across modern multi-core systems.

2. **NUMA Awareness**: Deep integration with NUMA topology ensures optimal memory locality and performance.

3. **Fragmentation Management**: Migration types and compaction work together to minimize long-term fragmentation.

4. **Memory Pressure Handling**: Sophisticated watermark and reclaim integration prevents OOM conditions while maintaining performance.

5. **Debug and Monitoring**: Comprehensive debugging facilities enable effective troubleshooting and system optimization.

6. **Extensibility**: Clean integration points allow for features like CMA, memory hotplug, and cgroup accounting.

The implementation demonstrates decades of evolution and optimization, making it one of the most critical and well-engineered components of the Linux kernel.