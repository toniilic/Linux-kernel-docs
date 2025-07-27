# lib/timerqueue.c - Generic Timer Queue Implementation

## Overview

This file implements a generic timer queue data structure for the Linux kernel. It provides an efficient mechanism for managing timers ordered by their expiration times, using red-black trees for optimal performance in insertion, deletion, and finding the earliest timer.

## Purpose and Design Goals

### Primary Purpose
- **Timer Management**: Maintain a sorted queue of timers ordered by expiration time
- **Efficient Operations**: Fast insertion, deletion, and early expiration lookup
- **Generic Interface**: Reusable across different kernel subsystems

### Design Principles
- **Performance**: O(log n) operations for add/remove, O(1) for earliest timer
- **Simplicity**: Minimal, focused API with clear semantics
- **Safety**: Built-in consistency checks and debugging support
- **Thread Safety**: Designed for external serialization (no internal locking)

## Core Data Structures

### `struct timerqueue_head`
```c
struct timerqueue_head {
    struct rb_root_cached rb_root;
};
```
- **Purpose**: Container for the timer queue
- **Implementation**: Uses cached red-black tree for O(1) leftmost access
- **Caching**: Maintains pointer to earliest (leftmost) timer for fast access

### `struct timerqueue_node`
```c
struct timerqueue_node {
    struct rb_node node;
    ktime_t expires;
};
```
- **Purpose**: Individual timer entry in the queue
- **Fields**:
  - `node`: Red-black tree linkage
  - `expires`: Expiration time (ktime_t for high precision)

## Key Functions

### `timerqueue_add()` - Add Timer to Queue
```c
bool timerqueue_add(struct timerqueue_head *head, struct timerqueue_node *node)
```

**Purpose**: Inserts a timer into the queue in sorted order by expiration time

**Parameters**:
- `head`: Pointer to the timer queue head
- `node`: Timer node to be added (must not already be in a queue)

**Algorithm**:
1. **Validation**: Ensures node is not already in a tree (`WARN_ON_ONCE`)
2. **Insertion**: Uses `rb_add_cached()` with custom comparison function
3. **Ordering**: Maintains sorted order by expiration time
4. **Caching**: Updates cached leftmost pointer if necessary

**Return Value**:
- `true`: If the added timer is now the earliest expiring timer
- `false`: If other timers expire before this one

**Usage Pattern**:
```c
struct timerqueue_head head;
struct timerqueue_node timer;

timer.expires = ktime_add_ns(ktime_get(), 1000000); // 1ms from now
bool is_earliest = timerqueue_add(&head, &timer);
if (is_earliest) {
    // This timer is now the next to expire
    reprogram_hardware_timer();
}
```

### `timerqueue_del()` - Remove Timer from Queue
```c
bool timerqueue_del(struct timerqueue_head *head, struct timerqueue_node *node)
```

**Purpose**: Removes a timer from the queue

**Parameters**:
- `head`: Pointer to the timer queue head
- `node`: Timer node to be removed (must be in the queue)

**Algorithm**:
1. **Validation**: Ensures node is actually in a tree (`WARN_ON_ONCE`)
2. **Removal**: Uses `rb_erase_cached()` to remove from tree
3. **Cleanup**: Clears the node's tree linkage (`RB_CLEAR_NODE`)
4. **State Check**: Determines if queue is empty after removal

**Return Value**:
- `true`: If the queue still contains timers after removal
- `false`: If the queue is now empty

**Usage Pattern**:
```c
bool queue_not_empty = timerqueue_del(&head, &timer);
if (!queue_not_empty) {
    // No more timers, can disable hardware timer
    disable_hardware_timer();
}
```

### `timerqueue_iterate_next()` - Iterator Function
```c
struct timerqueue_node *timerqueue_iterate_next(struct timerqueue_node *node)
```

**Purpose**: Provides safe iteration through the timer queue

**Parameters**:
- `node`: Current timer node (NULL for starting iteration)

**Return Value**:
- Pointer to the next timer node in expiration order
- `NULL` if no more timers or invalid input

**Algorithm**:
1. **Null Check**: Handles NULL input gracefully
2. **Tree Navigation**: Uses `rb_next()` to find next node in sorted order
3. **Container Conversion**: Converts rb_node back to timerqueue_node

**Usage Pattern**:
```c
struct timerqueue_node *timer;
struct timerqueue_node *first = timerqueue_getnext(&head);

// Iterate through all timers
for (timer = first; timer; timer = timerqueue_iterate_next(timer)) {
    printk("Timer expires at: %lld\n", timer->expires);
}
```

## Supporting Infrastructure

### `__node_2_tq()` Macro
```c
#define __node_2_tq(_n) rb_entry((_n), struct timerqueue_node, node)
```
- **Purpose**: Converts rb_node pointer to timerqueue_node pointer
- **Type Safety**: Uses kernel's type-safe container_of mechanism
- **Internal Use**: Helper for other functions in the file

### `__timerqueue_less()` - Comparison Function
```c
static inline bool __timerqueue_less(struct rb_node *a, const struct rb_node *b)
```
- **Purpose**: Comparison function for red-black tree ordering
- **Criteria**: Orders by expiration time (`expires` field)
- **Result**: Returns true if timer `a` expires before timer `b`
- **Used By**: `rb_add_cached()` for proper tree insertion

## Red-Black Tree Integration

### Why Red-Black Trees?
- **Balanced**: Guarantees O(log n) operations
- **Cache Efficiency**: Cached variant provides O(1) leftmost access
- **Memory Efficient**: No additional storage overhead
- **Proven**: Well-tested data structure in kernel

### Cached Red-Black Tree Benefits
- **Fast Earliest Access**: O(1) access to next expiring timer
- **Efficient Updates**: Automatic cache maintenance during modifications
- **Optimal for Timers**: Perfect match for timer queue access patterns

### Tree Properties Maintained
- **Ordering**: Left subtree expires before right subtree
- **Balance**: Tree remains approximately balanced
- **Caching**: Leftmost (earliest) node always cached

## Memory Management and Lifecycle

### Node Initialization
```c
// Nodes must be initialized before use
struct timerqueue_node timer;
RB_CLEAR_NODE(&timer.node);  // Mark as not-in-tree
timer.expires = target_time;
```

### State Tracking
- **Empty Nodes**: `RB_EMPTY_NODE()` checks if node is in a tree
- **Empty Trees**: `RB_EMPTY_ROOT()` checks if tree has any nodes
- **State Consistency**: Automatic state management during add/del

### Memory Safety
- **No Dynamic Allocation**: Nodes embedded in containing structures
- **Lifetime Management**: Caller responsible for node lifetime
- **Corruption Prevention**: Consistency checks prevent double-add/remove

## Performance Characteristics

### Time Complexity
- **Add**: O(log n) - tree insertion with comparison
- **Delete**: O(log n) - tree removal and rebalancing
- **Get Next**: O(1) - cached leftmost access
- **Iterate**: O(1) per step - simple tree traversal

### Space Complexity
- **Per Node**: One rb_node + one ktime_t (typically 24-32 bytes)
- **Per Queue**: One rb_root_cached (typically 8-16 bytes)
- **No Overhead**: No additional metadata or auxiliary structures

### Cache Performance
- **Sequential Access**: Iteration follows tree structure (reasonable locality)
- **Early Timer Access**: Cached leftmost provides optimal access
- **Modification Locality**: Tree operations work on small subtrees

## Thread Safety and Locking

### External Serialization Required
The timer queue implementation provides **no internal locking**:

```c
/*
 * NOTE: All of the following functions need to be serialized
 * to avoid races. No locking is done by this library code.
 */
```

### Typical Usage Pattern
```c
static DEFINE_SPINLOCK(timer_lock);

void add_timer_safe(struct timerqueue_head *head, struct timerqueue_node *node) {
    unsigned long flags;
    
    spin_lock_irqsave(&timer_lock, flags);
    timerqueue_add(head, node);
    spin_unlock_irqrestore(&timer_lock, flags);
}
```

### Rationale for External Locking
- **Flexibility**: Allows callers to choose appropriate locking strategy
- **Performance**: Avoids lock overhead in single-threaded contexts
- **Composition**: Enables atomic multi-operation sequences
- **Context Awareness**: Caller knows interrupt/scheduling context

## Error Handling and Debugging

### Consistency Checks
```c
WARN_ON_ONCE(!RB_EMPTY_NODE(&node->node));  // Detect double-add
WARN_ON_ONCE(RB_EMPTY_NODE(&node->node));   // Detect invalid remove
```

### Debug Support
- **State Validation**: Automatic checks for node state consistency
- **One-Time Warnings**: `WARN_ON_ONCE` prevents log spam
- **Tree Integrity**: Red-black tree code includes extensive validation

### Common Error Conditions
- **Double Add**: Adding a node already in a queue
- **Invalid Remove**: Removing a node not in any queue
- **Use After Free**: Accessing freed timer nodes
- **Uninitialized Nodes**: Using nodes without proper initialization

## Integration with Kernel Subsystems

### High-Resolution Timers (hrtimers)
```c
struct hrtimer {
    struct timerqueue_node node;
    // ... other fields
};
```

### CPU Scheduler
- **Load balancing timers**: Migration deadlines
- **Bandwidth throttling**: CFS and RT throttling timers
- **Periodic tasks**: Various scheduler maintenance timers

### Device Drivers
- **Timeout Handling**: I/O operation deadlines
- **Periodic Operations**: Regular device maintenance
- **Rate Limiting**: Bandwidth control and throttling

### Networking Stack
- **Protocol Timers**: TCP retransmission, keepalive timers
- **Quality of Service**: Traffic shaping and rate limiting
- **Connection Management**: Timeout detection and cleanup

## Usage Examples

### Basic Timer Queue Operations
```c
struct timerqueue_head timer_queue;
struct timerqueue_node timer1, timer2;

// Initialize queue
timer_queue.rb_root = RB_ROOT_CACHED;

// Initialize and add timers
RB_CLEAR_NODE(&timer1.node);
timer1.expires = ktime_add_ms(ktime_get(), 100);  // 100ms from now
timerqueue_add(&timer_queue, &timer1);

RB_CLEAR_NODE(&timer2.node);
timer2.expires = ktime_add_ms(ktime_get(), 50);   // 50ms from now
bool is_earliest = timerqueue_add(&timer_queue, &timer2);
// is_earliest will be true since timer2 expires first

// Get earliest timer
struct timerqueue_node *next = timerqueue_getnext(&timer_queue);
// next points to timer2

// Remove timer
timerqueue_del(&timer_queue, &timer2);
```

### Integration with Higher-Level Timer Systems
```c
struct my_timer {
    struct timerqueue_node tq_node;
    void (*callback)(struct my_timer *);
    void *data;
};

void schedule_my_timer(struct my_timer *timer, ktime_t expires) {
    timer->tq_node.expires = expires;
    
    spin_lock(&timer_queue_lock);
    bool reprogramNeeded = timerqueue_add(&my_timer_queue, &timer->tq_node);
    if (reprogramNeeded) {
        program_hardware_timer(expires);
    }
    spin_unlock(&timer_queue_lock);
}
```

This implementation provides a robust, efficient foundation for timer management in the Linux kernel, enabling precise timing operations with minimal overhead while maintaining the flexibility needed for diverse kernel subsystems.