# Linux Kernel IDR/IDA Implementation Documentation

## Executive Summary

The Linux kernel's IDR (ID Allocator) and IDA (ID Allocator Array) provide efficient mechanisms for allocating and managing unique integer identifiers. IDR associates IDs with pointer values while IDA allocates IDs without pointer storage, optimizing for memory efficiency. Both are built on top of the XArray radix tree implementation, providing scalable O(log n) operations with sophisticated memory management and concurrency support.

---

## Agent 1: Core IDR Data Structures and Radix Tree Foundation

### Data Structure Architecture

#### IDR Structure
```c
struct idr {
    struct radix_tree_root idr_rt;  // XArray-based radix tree
    unsigned int idr_base;          // Base offset for ID mapping
    unsigned int idr_next;          // Next ID for cyclic allocation
};
```

**Key Design Elements:**
- **Radix Tree Foundation**: Built on XArray, a sophisticated radix tree implementation
- **Base Offset Support**: `idr_base` allows custom starting IDs (e.g., starting from 1 instead of 0)
- **Cyclic Allocation State**: `idr_next` tracks position for `idr_alloc_cyclic()`

#### IDA Structure
```c
struct ida {
    struct xarray xa;  // Direct XArray usage
};

struct ida_bitmap {
    unsigned long bitmap[IDA_BITMAP_LONGS];  // 128-byte bitmap chunks
};
```

**Memory Optimization Strategy:**
- **Two-Tier Approach**: Value entries for sparse allocation, bitmap entries for dense allocation
- **Chunk Size**: 128-byte bitmaps (1024 bits) for efficient memory usage
- **Automatic Scaling**: Seamless conversion between value and bitmap entries

### Radix Tree Foundation

#### Tree Organization
- **Branching Factor**: 64-way branching (6 bits per level on 64-bit systems)
- **Height Management**: Dynamic tree extension based on maximum index
- **Node Structure**: Internal nodes contain slots and tag arrays
- **Leaf Storage**: Direct pointer storage for IDR, bitmap/value storage for IDA

#### Tag System
```c
#define IDR_FREE 0  // Tag 0 tracks free slots
```

**Tag Usage:**
- **Free Space Tracking**: IDR_FREE tag marks nodes with available slots
- **Propagation**: Tags bubble up from leaves to root for efficient searching
- **Search Optimization**: Enables O(log n) free slot discovery

#### Tree Traversal Mechanisms
```c
struct radix_tree_iter {
    unsigned long index;        // Current position
    unsigned long next_index;   // End of current chunk
    unsigned long tags;         // Tag mask for iteration
    struct radix_tree_node *node; // Current node
};
```

**Traversal Features:**
- **Chunk-Based Iteration**: Processes multiple slots per iteration
- **Tagged Iteration**: Efficient iteration over tagged (free) slots
- **RCU-Safe Traversal**: Lockless reads with proper memory ordering

---

## Agent 2: ID Allocation Algorithms and Strategies

### Allocation Strategies

#### First-Fit Allocation (IDR)
**Algorithm**: `idr_get_free()` function
```c
void __rcu **idr_get_free(struct radix_tree_root *root,
                         struct radix_tree_iter *iter, gfp_t gfp,
                         unsigned long max)
```

**Process Flow:**
1. **Tree Extension Check**: Extend tree if start > maxindex
2. **Free Slot Search**: Use IDR_FREE tags to locate available slots
3. **Node Allocation**: Create intermediate nodes as needed
4. **Tag Management**: Clear IDR_FREE when slot becomes occupied

**Optimization Features:**
- **Tag-Guided Search**: Follows IDR_FREE tags down the tree
- **Backtracking**: When no free slots found, backtrack to parent level
- **Growth on Demand**: Tree extends only when needed

#### Bitmap-Based Allocation (IDA)
**Two-Stage Strategy:**

**Stage 1: Value Entry Allocation**
```c
if (xa_is_value(bitmap)) {
    unsigned long tmp = xa_to_value(bitmap);
    bit = find_next_zero_bit(&tmp, BITS_PER_XA_VALUE, bit);
    // Use up to 63 bits directly in XArray slot
}
```

**Stage 2: Bitmap Entry Allocation**
```c
struct ida_bitmap *bitmap = kzalloc(sizeof(*bitmap), gfp);
bit = find_next_zero_bit(bitmap->bitmap, IDA_BITMAP_BITS, bit);
__set_bit(bit, bitmap->bitmap);
```

### Range Management

#### IDR Range Handling
- **Base Adjustment**: All operations adjust for `idr_base` offset
- **Signed Integer Support**: Proper handling of INT_MAX boundaries
- **Overflow Protection**: Bounds checking prevents wraparound

#### IDA Range Optimization
- **Chunk Alignment**: IDA divides index space by IDA_BITMAP_BITS (1024)
- **Bit-Level Precision**: Fine-grained allocation within chunks
- **Range Spanning**: Handles allocations across multiple bitmap chunks

### Cyclic Allocation Strategy

**Algorithm Benefits:**
- **Load Distribution**: Prevents clustering in low-numbered IDs
- **Aging Behavior**: Recently freed IDs are reused less immediately
- **Wraparound Logic**: Seamless transition from end to beginning of range

**Implementation:**
```c
int idr_alloc_cyclic(struct idr *idr, void *ptr, int start, int end, gfp_t gfp)
{
    u32 id = idr->idr_next;  // Start from last allocation
    // Try allocation from current position
    err = idr_alloc_u32(idr, ptr, &id, max, gfp);
    if (err == -ENOSPC && id > start) {
        // Wrap around and try from beginning
        id = start;
        err = idr_alloc_u32(idr, ptr, &id, max, gfp);
    }
    idr->idr_next = id + 1;  // Update for next allocation
}
```

### Fragmentation Handling

#### IDR Fragmentation
- **Tree Compaction**: Unused internal nodes can be garbage collected
- **Tag Optimization**: IDR_FREE tags enable skipping over allocated regions
- **Memory Reclaim**: Node deallocation when subtrees become empty

#### IDA Fragmentation
- **Value Entry Transition**: Small gaps use value entries (no bitmap overhead)
- **Bitmap Consolidation**: Multiple sparse value entries can merge into bitmaps
- **Free Mark Management**: XA_FREE_MARK tracks partially-filled bitmaps

---

## Agent 3: ID Lookup and Removal Operations

### Lookup Algorithms

#### IDR Lookup Implementation
```c
void *idr_find(const struct idr *idr, unsigned long id)
{
    return radix_tree_lookup(&idr->idr_rt, id - idr->idr_base);
}
```

**Performance Characteristics:**
- **O(log n) Complexity**: Logarithmic lookup time
- **RCU-Safe**: No locking required for read operations
- **Cache Efficiency**: Radix tree structure optimizes memory locality

#### Lookup Optimization Techniques
- **Path Compression**: Direct traversal to target node
- **Prefetching**: Node layout optimizes cache line usage
- **Branching Minimization**: High-fanout nodes reduce tree height

### Removal Strategies

#### IDR Removal Process
```c
void *idr_remove(struct idr *idr, unsigned long id)
{
    return radix_tree_delete_item(&idr->idr_rt, id - idr->idr_base, NULL);
}
```

**Removal Steps:**
1. **Locate Entry**: Navigate to target slot using index
2. **Extract Value**: Retrieve stored pointer before deletion
3. **Clear Slot**: Set slot to NULL (marking as free)
4. **Update Tags**: Set IDR_FREE tag for freed slot
5. **Propagate Tags**: Bubble tag changes up to parent nodes
6. **Node Cleanup**: Remove empty internal nodes

#### Tree Maintenance During Removal

**Tag Propagation Algorithm:**
```c
static void tag_set(struct radix_tree_node *node, unsigned int tag, int offset)
{
    __set_bit(offset, node->tags[tag]);
    // Tag automatically propagates to parent during tree operations
}
```

**Node Cleanup Strategy:**
- **Reference Counting**: Track child node count in each internal node
- **Lazy Deletion**: Defer node deallocation until safe
- **Memory Barriers**: Proper ordering for concurrent readers

#### IDA Removal Implementation
```c
void ida_free(struct ida *ida, unsigned int id)
{
    unsigned bit = id % IDA_BITMAP_BITS;
    // Handle both value entries and bitmap entries
    if (xa_is_value(bitmap)) {
        // Clear bit in value entry
        v &= ~(1UL << bit);
    } else {
        // Clear bit in bitmap entry
        __clear_bit(bit, bitmap->bitmap);
        if (bitmap_empty(bitmap->bitmap, IDA_BITMAP_BITS)) {
            kfree(bitmap);  // Free empty bitmap
        }
    }
}
```

### Tree Balancing and Maintenance

#### Automatic Rebalancing
- **Height Adjustment**: Tree contracts when maximum index decreases
- **Node Merging**: Adjacent sparse nodes can be consolidated
- **Memory Optimization**: Unused levels are eliminated

#### Integrity Maintenance
- **Double-Free Detection**: IDA warns when freeing unallocated IDs
- **Consistency Checks**: Debug builds verify tree structure
- **Tag Validation**: Ensures tag state matches actual free slots

### Iteration and Enumeration

#### IDR Iteration Patterns
```c
int idr_for_each(const struct idr *idr,
                int (*fn)(int id, void *p, void *data), void *data)
{
    radix_tree_for_each_slot(slot, &idr->idr_rt, &iter, 0) {
        ret = fn(id, rcu_dereference_raw(*slot), data);
        if (ret) return ret;
    }
}
```

**Iteration Features:**
- **Stable Iteration**: Concurrent modifications don't skip/duplicate entries
- **Early Termination**: Callback can halt iteration by returning non-zero
- **RCU Protection**: Safe for lockless enumeration

#### Advanced Lookup Operations
```c
void *idr_get_next_ul(struct idr *idr, unsigned long *nextid);
void *idr_replace(struct idr *idr, void *ptr, unsigned long id);
```

**Use Cases:**
- **Range Scanning**: Find next allocated ID after given position
- **Atomic Updates**: Replace pointer value without removing ID
- **Gap Analysis**: Identify unallocated ranges

---

## Agent 4: Memory Management and Performance Optimizations

### Memory Efficiency Strategies

#### IDA Memory Optimization
**Value Entry Strategy:**
- **Threshold**: Use value entries for ≤63 allocated bits per chunk
- **Embedding**: Store bitmap directly in XArray slot (8 bytes)
- **Automatic Promotion**: Convert to full bitmap when density increases

**Bitmap Memory Management:**
```c
#define IDA_CHUNK_SIZE 128        // 128 bytes per bitmap
#define IDA_BITMAP_LONGS (IDA_CHUNK_SIZE / sizeof(long))
#define IDA_BITMAP_BITS (IDA_BITMAP_LONGS * sizeof(long) * 8)  // 1024 bits
```

**Memory Efficiency Comparison:**
- **IDR**: ~8 bytes per ID (pointer storage)
- **IDA**: ~0.125 bytes per ID (1 bit per ID in dense regions)
- **IDA Sparse**: 0 bytes for unallocated ranges (value entries)

#### Cache Optimization

**Node Layout Optimization:**
- **Cache Line Alignment**: Radix tree nodes aligned for optimal cache usage
- **Prefetch Patterns**: Sequential slot access patterns leverage hardware prefetching
- **Working Set Locality**: Hot nodes remain in cache during burst allocations

**Memory Pool Management:**
```c
DEFINE_PER_CPU(struct radix_tree_preload, radix_tree_preloads);
#define RADIX_TREE_PRELOAD_SIZE (RADIX_TREE_MAX_PATH * 2 - 1)
```

**Preallocation Benefits:**
- **Atomic Context Support**: Pre-allocated nodes for IRQ contexts
- **Allocation Latency**: Reduced allocation time in critical paths
- **Memory Fragmentation**: Better memory layout through batched allocation

### Scalability Optimizations

#### Concurrent Access Patterns

**IDR Concurrency Model:**
- **External Locking**: Caller-managed synchronization for modifications
- **RCU Read Access**: Lockless lookups with memory ordering guarantees
- **Write Exclusion**: Modifications require exclusive access

**IDA Internal Locking:**
```c
XA_STATE(xas, &ida->xa, min / IDA_BITMAP_BITS);
xas_lock_irqsave(&xas, flags);
// Critical section with interrupt protection
xas_unlock_irqrestore(&xas, flags);
```

**Lock Granularity:**
- **Per-IDA Locking**: Each IDA has independent locking
- **IRQ Safety**: Interrupt-safe spinlocks prevent deadlocks
- **Short Critical Sections**: Minimal lock hold times

#### NUMA Considerations

**Memory Locality:**
- **Node Affinity**: Radix tree nodes allocated on local NUMA node
- **Access Patterns**: Frequently accessed nodes migrate to active NUMA zones
- **Distributed Workloads**: Per-CPU preallocation reduces NUMA traffic

### Performance Characteristics

#### Time Complexity Analysis
- **Allocation**: O(log n) where n is maximum allocated ID
- **Lookup**: O(log n) radix tree traversal
- **Removal**: O(log n) plus tag propagation overhead
- **Iteration**: O(k) where k is number of allocated IDs

#### Space Complexity
- **IDR Overhead**: O(n) storage for n allocated IDs
- **IDA Overhead**: O(n/1024) for dense allocation, O(k) for sparse (k = allocated IDs)
- **Tree Structure**: O(log n) internal nodes

#### Benchmark Performance Metrics
- **Allocation Rate**: ~1M allocations/second per core (typical workload)
- **Lookup Latency**: ~50-100ns per lookup (cache-hot scenario)
- **Memory Overhead**: IDR: 100% overhead, IDA: ~12.5% overhead (dense), ~0% overhead (sparse)

### Memory Reclamation

#### Garbage Collection Strategy
```c
void idr_destroy(struct idr *idr)
{
    struct radix_tree_node *node = rcu_dereference_raw(idr->idr_rt.xa_head);
    if (radix_tree_is_internal_node(node))
        radix_tree_free_nodes(node);  // Recursive node freeing
}
```

**Cleanup Features:**
- **Complete Teardown**: Frees all internal nodes and bitmaps
- **RCU Grace Periods**: Delayed freeing for concurrent readers
- **Memory Leak Prevention**: Ensures no internal memory leaks

#### Dynamic Memory Management
- **On-Demand Allocation**: Nodes allocated only when needed
- **Automatic Shrinking**: Tree contracts as IDs are freed
- **Memory Pressure Response**: Cooperates with kernel memory management

---

## Agent 5: API Design and Kernel Integration Patterns

### API Design Philosophy

#### Consistency Principles
**Naming Conventions:**
- **idr_**: Functions operating on IDR structures
- **ida_**: Functions operating on IDA structures  
- **Suffix Patterns**: `_alloc`, `_free`, `_find`, `_destroy` for common operations

**Return Value Conventions:**
- **Allocation Functions**: Return allocated ID (≥0) or negative error code
- **Lookup Functions**: Return found pointer or NULL
- **Void Functions**: Operations that cannot fail (e.g., `ida_free`)

#### Memory Allocation Integration
```c
// GFP flag propagation through all allocation paths
int idr_alloc(struct idr *idr, void *ptr, int start, int end, gfp_t gfp);
int ida_alloc_range(struct ida *ida, unsigned int min, unsigned int max, gfp_t gfp);
```

**GFP Flag Handling:**
- **Context Awareness**: Respects atomic vs. sleeping contexts
- **Fallback Strategies**: Internal GFP_NOWAIT usage in atomic sections
- **Error Propagation**: Memory allocation failures properly reported

### Kernel Subsystem Integration

#### Common Usage Patterns

**Device ID Management:**
```c
// Example: Network device allocation
static DEFINE_IDA(net_dev_ida);

int register_netdev(struct net_device *dev) {
    int id = ida_alloc(&net_dev_ida, GFP_KERNEL);
    if (id < 0) return id;
    dev->ifindex = id;
    // ... register device ...
}
```

**Process ID Management:**
```c
// Example: PID allocation in kernel/pid.c
struct pid_namespace {
    struct idr idr;  // IDR for PID -> struct pid mapping
    // ...
};

struct pid *alloc_pid(struct pid_namespace *ns) {
    int pid = idr_alloc_cyclic(&ns->idr, NULL, pid_min, pid_max, GFP_KERNEL);
    // ...
}
```

#### Integration Patterns

**Resource Management Pattern:**
1. **DEFINE_IDR/DEFINE_IDA**: Static initialization
2. **Dynamic Initialization**: `idr_init()` or `ida_init()` for embedded structures
3. **Allocation**: Context-appropriate GFP flags
4. **Error Handling**: Proper cleanup on allocation failure
5. **Cleanup**: `idr_destroy()` or `ida_destroy()` on subsystem shutdown

**Locking Integration:**
```c
// Common pattern for IDR usage
DEFINE_IDR(my_idr);
static DEFINE_SPINLOCK(my_lock);

int allocate_resource(struct my_resource *res) {
    int id, err;
    
    idr_preload(GFP_KERNEL);
    spin_lock(&my_lock);
    id = idr_alloc(&my_idr, res, 0, 0, GFP_NOWAIT);
    spin_unlock(&my_lock);
    idr_preload_end();
    
    return id;
}
```

### Error Handling Patterns

#### Comprehensive Error Codes
```c
// Standard error returns
-ENOMEM    // Memory allocation failure
-ENOSPC    // No free IDs in range
-ENOENT    // ID not found (lookup/replace operations)
-EINVAL    // Invalid parameters
```

#### Error Handling Best Practices
```c
// Robust allocation pattern
int err = ida_alloc_range(&my_ida, min, max, GFP_KERNEL);
if (err < 0) {
    switch (err) {
    case -ENOMEM:
        return -ENOMEM;  // Propagate memory pressure
    case -ENOSPC:
        return -EAGAIN;  // Suggest retry later
    default:
        return err;      // Unexpected error
    }
}
int id = err;  // Success: err contains allocated ID
```

#### Debug and Validation Support
```c
// IDA double-free detection
void ida_free(struct ida *ida, unsigned int id) {
    // ...
    if (!test_bit(bit, bitmap->bitmap))
        goto err;
    // ...
err:
    WARN(1, "ida_free called for id=%d which is not allocated.\n", id);
}
```

### Advanced API Features

#### Iterator Support
```c
// Comprehensive iteration macros
#define idr_for_each_entry(idr, entry, id) \
    for (id = 0; ((entry) = idr_get_next(idr, &(id))) != NULL; id += 1U)

#define idr_for_each_entry_continue(idr, entry, id) \
    for ((entry) = idr_get_next((idr), &(id)); \
         entry; \
         ++id, (entry) = idr_get_next((idr), &(id)))
```

**Iterator Features:**
- **Stable Iteration**: Safe under concurrent modifications
- **Continuation Support**: Resume iteration from arbitrary position
- **Range Iteration**: Iterate over ID subranges

#### Atomic Operations Support
```c
// Class-based automatic cleanup (modern kernel pattern)
DEFINE_CLASS(idr_alloc, struct __class_idr,
             if (_T.id >= 0) idr_remove(_T.idr, _T.id),
             ((struct __class_idr){
                 .idr = idr,
                 .id = idr_alloc(idr, ptr, start, end, gfp),
             }),
             struct idr *idr, void *ptr, int start, int end, gfp_t gfp);
```

**Modern C++ Style Features:**
- **RAII Support**: Automatic cleanup using cleanup attributes
- **Scope-Based Management**: Resources freed at scope exit
- **Exception Safety**: Guaranteed cleanup even with early returns

---

## Concurrency Handling and Thread Safety Considerations

### Locking Models

#### IDR Locking Requirements
**Writer Synchronization:**
- **External Locking**: Callers must serialize all modifications
- **Lock Scope**: Must cover allocation, removal, and replacement operations
- **RCU Grace Periods**: Required for safe memory reclamation

**Reader Synchronization:**
```c
// RCU-protected read pattern
rcu_read_lock();
ptr = idr_find(&my_idr, id);
if (ptr) {
    // Use ptr safely within RCU critical section
    process_object(ptr);
}
rcu_read_unlock();
```

#### IDA Thread Safety
**Internal Locking:**
- **XArray State Protection**: XA_STATE provides built-in locking
- **IRQ Safety**: `xas_lock_irqsave()` prevents interrupt deadlocks
- **Lockless Reads**: Some operations like `ida_is_empty()` are lockless

### Memory Ordering and Barriers

#### RCU Memory Ordering
```c
// IDR allocation with proper memory barriers
radix_tree_iter_replace(&idr->idr_rt, &iter, slot, ptr);
// Memory barrier inside radix_tree_iter_replace() ensures:
// 1. ptr is fully initialized before assignment
// 2. Assignment is visible to concurrent readers
```

#### XArray Memory Guarantees
- **Store Ordering**: XArray operations include necessary memory barriers
- **Load Ordering**: RCU dereference provides proper load barriers
- **Visibility**: Changes become visible to concurrent readers atomically

### Concurrent Access Patterns

#### Read-Heavy Workloads
**Optimization Strategies:**
- **RCU Lookup**: O(log n) lockless reads
- **Cache Locality**: Hot entries remain in CPU caches
- **No Lock Contention**: Multiple readers proceed in parallel

#### Write-Heavy Workloads
**Performance Considerations:**
- **Lock Granularity**: Per-IDR/IDA locking minimizes contention
- **Preallocation**: Reduces allocation latency in critical sections
- **Batch Operations**: Group allocations to amortize lock overhead

#### Mixed Workloads
**Balancing Strategies:**
- **Reader Priority**: RCU allows reads during write operations
- **Writer Batching**: Group multiple modifications under single lock
- **Lock-Free Paths**: Some operations proceed without blocking readers

---

## Usage Examples and Common Patterns in Kernel Code

### Basic Usage Examples

#### Simple IDR Usage
```c
#include <linux/idr.h>

static DEFINE_IDR(connection_idr);
static DEFINE_SPINLOCK(connection_lock);

struct connection {
    int id;
    struct socket *sock;
    // ... other fields
};

// Allocate new connection with unique ID
int create_connection(struct socket *sock) {
    struct connection *conn;
    int id, err;
    
    conn = kzalloc(sizeof(*conn), GFP_KERNEL);
    if (!conn)
        return -ENOMEM;
    
    conn->sock = sock;
    
    idr_preload(GFP_KERNEL);
    spin_lock(&connection_lock);
    id = idr_alloc(&connection_idr, conn, 1, 0, GFP_NOWAIT);
    spin_unlock(&connection_lock);
    idr_preload_end();
    
    if (id < 0) {
        kfree(conn);
        return id;
    }
    
    conn->id = id;
    return id;
}

// Find connection by ID
struct connection *find_connection(int id) {
    struct connection *conn;
    
    rcu_read_lock();
    conn = idr_find(&connection_idr, id);
    rcu_read_unlock();
    
    return conn;
}

// Remove connection
void destroy_connection(int id) {
    struct connection *conn;
    
    spin_lock(&connection_lock);
    conn = idr_remove(&connection_idr, id);
    spin_unlock(&connection_lock);
    
    if (conn) {
        synchronize_rcu();  // Wait for readers
        kfree(conn);
    }
}
```

#### Simple IDA Usage
```c
#include <linux/idr.h>

static DEFINE_IDA(device_ida);

// Allocate device number
int alloc_device_number(void) {
    return ida_alloc_range(&device_ida, 1, 65535, GFP_KERNEL);
}

// Free device number
void free_device_number(int devno) {
    ida_free(&device_ida, devno);
}

// Check if device number is allocated
bool is_device_number_allocated(int devno) {
    return ida_exists(&device_ida, devno);
}
```

### Advanced Integration Examples

#### Network Interface Management
```c
// From net/core/dev.c (simplified)
static DEFINE_IDA(netdev_ida);

int dev_new_index(struct net *net) {
    int ifindex;
    
    for (;;) {
        ifindex = ida_alloc_range(&netdev_ida, 1, INT_MAX, GFP_KERNEL);
        if (ifindex < 0)
            return ifindex;
            
        // Verify uniqueness in namespace
        if (!__dev_get_by_index(net, ifindex))
            return ifindex;
            
        ida_free(&netdev_ida, ifindex);
        // Retry with next available index
    }
}

void dev_free_index(int ifindex) {
    ida_free(&netdev_ida, ifindex);
}
```

#### Process ID Management
```c
// From kernel/pid.c (simplified)
struct pid *alloc_pid(struct pid_namespace *ns) {
    struct pid *pid;
    enum pid_type type;
    int i, nr;
    struct pid_namespace *tmp;
    struct upid *upid;
    
    pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
    if (!pid)
        return ERR_PTR(-ENOMEM);
    
    tmp = ns;
    pid->level = ns->level;
    
    for (i = ns->level; i >= 0; i--) {
        int pid_min = 1;
        
        idr_preload(GFP_KERNEL);
        spin_lock_irq(&pidmap_lock);
        
        nr = idr_alloc_cyclic(&tmp->idr, NULL, pid_min,
                             pid_max_limit(tmp), GFP_NOWAIT);
        
        spin_unlock_irq(&pidmap_lock);
        idr_preload_end();
        
        if (nr < 0) {
            retval = (nr == -ENOSPC) ? -EAGAIN : nr;
            goto out_free;
        }
        
        pid->numbers[i].nr = nr;
        pid->numbers[i].ns = tmp;
        tmp = tmp->parent;
    }
    
    // ... initialize pid structure ...
    return pid;
}
```

### Error Handling Patterns

#### Robust Resource Management
```c
struct resource_manager {
    struct idr resources;
    spinlock_t lock;
    atomic_t refcount;
};

int resource_manager_alloc(struct resource_manager *mgr, 
                          struct resource *res) {
    int id, err = 0;
    
    if (!atomic_inc_not_zero(&mgr->refcount))
        return -ENODEV;  // Manager being destroyed
    
    idr_preload(GFP_KERNEL);
    spin_lock(&mgr->lock);
    
    id = idr_alloc(&mgr->resources, res, 0, 0, GFP_NOWAIT);
    if (id < 0) {
        err = id;
        goto unlock;
    }
    
    res->id = id;
    // Mark resource as allocated
    res->state = RESOURCE_ALLOCATED;
    
unlock:
    spin_unlock(&mgr->lock);
    idr_preload_end();
    
    if (err)
        atomic_dec(&mgr->refcount);
    
    return err ?: id;
}

void resource_manager_free(struct resource_manager *mgr, int id) {
    struct resource *res;
    
    spin_lock(&mgr->lock);
    res = idr_remove(&mgr->resources, id);
    spin_unlock(&mgr->lock);
    
    if (res) {
        res->state = RESOURCE_FREED;
        synchronize_rcu();
        kfree(res);
        atomic_dec(&mgr->refcount);
    }
}
```

#### Cleanup and Shutdown Patterns
```c
void resource_manager_destroy(struct resource_manager *mgr) {
    struct resource *res;
    int id;
    
    // Prevent new allocations
    atomic_set(&mgr->refcount, 0);
    
    // Free all allocated resources
    idr_for_each_entry(&mgr->resources, res, id) {
        res->state = RESOURCE_FREED;
        kfree(res);
    }
    
    // Clean up IDR structure
    idr_destroy(&mgr->resources);
    
    // Wait for any outstanding RCU readers
    synchronize_rcu();
}
```

### Performance Optimization Patterns

#### Batch Allocation
```c
int allocate_ids_batch(struct ida *ida, int *ids, size_t count) {
    size_t i;
    int err = 0;
    
    for (i = 0; i < count; i++) {
        ids[i] = ida_alloc(ida, GFP_KERNEL);
        if (ids[i] < 0) {
            err = ids[i];
            goto cleanup;
        }
    }
    
    return 0;
    
cleanup:
    // Free already allocated IDs
    while (i-- > 0)
        ida_free(ida, ids[i]);
    
    return err;
}
```

#### Cache-Friendly Iteration
```c
void process_all_connections(void (*callback)(struct connection *)) {
    struct connection *conn;
    int id;
    
    rcu_read_lock();
    idr_for_each_entry(&connection_idr, conn, id) {
        if (conn && conn->state == CONN_ACTIVE) {
            callback(conn);
        }
    }
    rcu_read_unlock();
}
```

These examples demonstrate real-world usage patterns found throughout the Linux kernel, showing how IDR and IDA integrate with various subsystems while maintaining proper locking, error handling, and resource management practices.

---

## Security Considerations and Best Practices

### Resource Exhaustion Protection
- **Range Limits**: Always specify reasonable maximum IDs to prevent exhaustion
- **Error Handling**: Gracefully handle -ENOSPC conditions
- **Resource Accounting**: Track allocated IDs for monitoring and limits

### Memory Safety
- **RCU Synchronization**: Proper grace periods before freeing objects
- **Double-Free Detection**: IDA automatically detects and warns on double-free
- **Bounds Checking**: All operations validate ID ranges

### Integration Guidelines
- **Locking Discipline**: Consistent locking patterns across subsystems
- **Error Propagation**: Meaningful error codes for debugging
- **Cleanup Ordering**: Proper shutdown sequences to prevent use-after-free

This comprehensive documentation covers the complete IDR/IDA implementation from multiple specialized perspectives, providing both theoretical understanding and practical guidance for kernel developers.