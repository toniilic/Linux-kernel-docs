# Linux Kernel Red-Black Tree Implementation

## Table of Contents
1. [Overview](#overview)
2. [Red-Black Tree Theory and Properties](#red-black-tree-theory-and-properties)
3. [Core Tree Operations and Balancing](#core-tree-operations-and-balancing)
4. [Augmented Tree Framework](#augmented-tree-framework)
5. [Performance Characteristics and Optimizations](#performance-characteristics-and-optimizations)
6. [Kernel Usage Patterns and Integration](#kernel-usage-patterns-and-integration)
7. [API Design and Programming Patterns](#api-design-and-programming-patterns)
8. [Implementation Details and Memory Efficiency](#implementation-details-and-memory-efficiency)
9. [Examples and Best Practices](#examples-and-best-practices)

## Overview

The Linux kernel's red-black tree implementation provides a self-balancing binary search tree data structure with guaranteed O(log n) performance for basic operations. Located primarily in `/lib/rbtree.c` and `/include/linux/rbtree*.h`, this implementation is optimized for kernel use with features including lockless lookups, augmented tree support, and memory-efficient node structures.

**Key Files:**
- `/lib/rbtree.c` - Core implementation
- `/include/linux/rbtree.h` - Main API and inline helpers
- `/include/linux/rbtree_augmented.h` - Augmented tree framework
- `/include/linux/rbtree_types.h` - Basic data structures
- `/include/linux/rbtree_latch.h` - Latched trees for RCU-safe access

## Red-Black Tree Theory and Properties

### Fundamental Properties

Red-black trees maintain five key invariants that ensure balanced structure:

```c
/*
 * red-black trees properties:  https://en.wikipedia.org/wiki/Rbtree
 *
 *  1) A node is either red or black
 *  2) The root is black  
 *  3) All leaves (NULL) are black
 *  4) Both children of every red node are black
 *  5) Every simple path from root to leaves contains the same number
 *     of black nodes.
 */
```

### Algorithmic Guarantees

The implementation provides **O(log n)** guarantees based on properties 4 and 5:

- Property 4 prevents consecutive red nodes
- Property 5 ensures uniform black node distribution
- Maximum path length is 2B (where B = number of black nodes)
- Tree height is guaranteed to be at most 2 * log₂(n + 1)

### Color Encoding

Colors are efficiently encoded using the least significant bit of the parent pointer:

```c
#define RB_RED    0
#define RB_BLACK  1

#define rb_color(rb)       __rb_color((rb)->__rb_parent_color)
#define rb_is_red(rb)      __rb_is_red((rb)->__rb_parent_color)
#define rb_is_black(rb)    __rb_is_black((rb)->__rb_parent_color)
```

## Core Tree Operations and Balancing

### Insertion Algorithm

The insertion process follows a two-phase approach:

1. **Standard BST insertion** using `rb_link_node()`
2. **Rebalancing** using `rb_insert_color()` or `__rb_insert()`

#### Insertion Cases

The balancing algorithm handles three main cases:

**Case 1: Red Uncle (Color Flips)**
```
       G            g
      / \          / \
     p   u  -->   P   U
    /            /
   n            n
```

**Case 2: Black Uncle + Triangle Configuration**
- Requires rotation to convert to Case 3
- Left-rotate at parent (if node is right child)
- Right-rotate at parent (if node is left child)

**Case 3: Black Uncle + Line Configuration**
```
        G           P
       / \         / \
      p   U  -->  n   g
     /                 \
    n                   U
```

### Deletion Algorithm

Deletion is more complex, implemented in `__rb_erase_augmented()`:

#### Deletion Cases

**Case 1: Red Sibling**
- Rotate to convert sibling to black
- Continue with subsequent cases

**Case 2: Black Sibling with Black Children**
- Color flip on sibling
- May require recursive fixing

**Case 3: Black Sibling with Red-Black Children**
- Rotation to prepare for Case 4

**Case 4: Black Sibling with Red Right Child**
- Final rotation and color adjustment

### Traversal Operations

```c
struct rb_node *rb_first(const struct rb_root *root);    // Leftmost node
struct rb_node *rb_last(const struct rb_root *root);     // Rightmost node
struct rb_node *rb_next(const struct rb_node *node);     // Inorder successor
struct rb_node *rb_prev(const struct rb_node *node);     // Inorder predecessor
```

**Postorder Traversal Support:**
```c
struct rb_node *rb_first_postorder(const struct rb_root *root);
struct rb_node *rb_next_postorder(const struct rb_node *node);

#define rbtree_postorder_for_each_entry_safe(pos, n, root, field) \
    for (pos = rb_entry_safe(rb_first_postorder(root), typeof(*pos), field); \
         pos && ({ n = rb_entry_safe(rb_next_postorder(&pos->field), \
                typeof(*pos), field); 1; }); \
         pos = n)
```

## Augmented Tree Framework

### Callback Structure

The augmented framework uses a callback interface for maintaining auxiliary data:

```c
struct rb_augment_callbacks {
    void (*propagate)(struct rb_node *node, struct rb_node *stop);
    void (*copy)(struct rb_node *old, struct rb_node *new);
    void (*rotate)(struct rb_node *old, struct rb_node *new);
};
```

### Template Macros

**Generic Augmentation:**
```c
#define RB_DECLARE_CALLBACKS(RBSTATIC, RBNAME, RBSTRUCT, RBFIELD, RBAUGMENTED, RBCOMPUTE)
```

**Maximum Value Augmentation:**
```c
#define RB_DECLARE_CALLBACKS_MAX(RBSTATIC, RBNAME, RBSTRUCT, RBFIELD, RBTYPE, RBAUGMENTED, RBCOMPUTE)
```

### Interval Trees

The interval tree implementation demonstrates augmented trees:

```c
#define INTERVAL_TREE_DEFINE(ITSTRUCT, ITRB, ITTYPE, ITSUBTREE, ITSTART, ITLAST, ITSTATIC, ITPREFIX)
```

**Key Features:**
- Maintains maximum endpoint in each subtree
- Efficient interval intersection queries
- O(log n + k) enumeration of intersecting intervals

## Performance Characteristics and Optimizations

### Memory Layout Optimizations

**Node Structure (24 bytes on 64-bit):**
```c
struct rb_node {
    unsigned long  __rb_parent_color;  // Parent pointer + color bit
    struct rb_node *rb_right;          // 8 bytes
    struct rb_node *rb_left;           // 8 bytes
} __attribute__((aligned(sizeof(long))));
```

**Space Efficiency:**
- Color stored in LSB of parent pointer (no extra storage)
- Alignment ensures low bits are available for flags
- Total overhead: 24 bytes per node on 64-bit systems

### Cached Trees for O(1) Minimum

```c
struct rb_root_cached {
    struct rb_root rb_root;
    struct rb_node *rb_leftmost;  // O(1) access to minimum
};

/* Same as rb_first(), but O(1) */
#define rb_first_cached(root) (root)->rb_leftmost
```

**Benefits:**
- Eliminates O(log n) traversal for minimum element
- Commonly needed operation in schedulers and timers
- Automatic maintenance during insertions/deletions

### Lockless Lookup Support

**Memory Ordering Guarantees:**
```c
/*
 * All stores to the tree structure (rb_left and rb_right) must be done using
 * WRITE_ONCE(). And we must not inadvertently cause (temporary) loops in the
 * tree structure as seen in program order.
 */
```

**Features:**
- `WRITE_ONCE()` for all pointer updates
- Prevents compiler reordering
- Enables lockless iteration (with caveats)
- RCU-safe variants available

### Compiler Optimizations

**Inlining Strategy:**
```c
static __always_inline void __rb_insert(...)
static __always_inline void ____rb_erase_color(...)
static __always_inline struct rb_node * __rb_erase_augmented(...)
```

**Branch Prediction:**
```c
if (unlikely(!parent)) {  // Root insertion case
    rb_set_parent_color(node, NULL, RB_BLACK);
    break;
}
```

## Kernel Usage Patterns and Integration

### Memory Management Integration

**VMA (Virtual Memory Area) Trees:**
- Track memory mappings in processes
- Fast lookup for page fault handling
- Interval-based operations for range queries

**Example Usage Pattern:**
```c
struct vm_area_struct {
    // ...
    struct rb_node vm_rb;
    // ...
};

static inline void vma_rb_insert(struct vm_area_struct *vma, 
                                struct rb_root *root)
{
    struct rb_node **link = &root->rb_node;
    struct rb_node *parent = NULL;
    
    while (*link) {
        parent = *link;
        if (vma->vm_start < rb_entry(parent, struct vm_area_struct, vm_rb)->vm_start)
            link = &parent->rb_left;
        else
            link = &parent->rb_right;
    }
    
    rb_link_node(&vma->vm_rb, parent, link);
    rb_insert_color(&vma->vm_rb, root);
}
```

### Networking Subsystem

**TCP Connection Tracking:**
```c
// From net/ipv4/tcp_input.c usage
static void tcp_store_ts_recent(struct tcp_sock *tp)
{
    // RB-tree used for timestamp tracking
}
```

**Packet Scheduling:**
- Fair queuing implementations
- Bandwidth management
- QoS enforcement

### Filesystem Integration

**File System Metadata:**
```c
// From fs/eventpoll.c
struct eventpoll {
    struct rb_root_cached rbr;  // Event items tree
    // ...
};
```

**Directory Entry Caching:**
- Fast pathname lookup
- Ordered iteration support
- Efficient range operations

## API Design and Programming Patterns

### Modern Helper Functions

The kernel provides high-level helpers that reduce boilerplate:

```c
// Simple insertion with comparison function
static __always_inline void
rb_add(struct rb_node *node, struct rb_root *tree,
       bool (*less)(struct rb_node *, const struct rb_node *))

// Find or insert pattern
static __always_inline struct rb_node *
rb_find_add(struct rb_node *node, struct rb_root *tree,
            int (*cmp)(struct rb_node *, const struct rb_node *))

// Simple lookup
static __always_inline struct rb_node *
rb_find(const void *key, const struct rb_root *tree,
        int (*cmp)(const void *key, const struct rb_node *))

// Range iteration
#define rb_for_each(node, key, tree, cmp) \
    for ((node) = rb_find_first((key), (tree), (cmp)); \
         (node); (node) = rb_next_match((key), (node), (cmp)))
```

### Container Integration Pattern

```c
#define rb_entry(ptr, type, member) container_of(ptr, type, member)

#define rb_entry_safe(ptr, type, member) \
    ({ typeof(ptr) ____ptr = (ptr); \
       ____ptr ? rb_entry(____ptr, type, member) : NULL; })
```

### Error Handling Patterns

**Empty Node Detection:**
```c
#define RB_EMPTY_NODE(node) \
    ((node)->__rb_parent_color == (unsigned long)(node))
#define RB_CLEAR_NODE(node) \
    ((node)->__rb_parent_color = (unsigned long)(node))
```

**Safe Tree Operations:**
```c
#define RB_EMPTY_ROOT(root) (READ_ONCE((root)->rb_node) == NULL)
```

## Implementation Details and Memory Efficiency

### Latched Trees for RCU

For environments requiring unconditional lockless access:

```c
struct latch_tree_root {
    seqcount_latch_t seq;
    struct rb_root tree[2];  // Double-buffered trees
};

static __always_inline struct latch_tree_node *
latch_tree_find(void *key, struct latch_tree_root *root,
                const struct latch_tree_ops *ops)
{
    struct latch_tree_node *node;
    unsigned int seq;
    
    do {
        seq = read_seqcount_latch(&root->seq);
        node = __lt_find(key, root, seq & 1, ops->comp);
    } while (read_seqcount_latch_retry(&root->seq, seq));
    
    return node;
}
```

### Dummy Callbacks Optimization

For non-augmented trees, the compiler optimizes away callback overhead:

```c
static inline void dummy_propagate(struct rb_node *node, struct rb_node *stop) {}
static inline void dummy_copy(struct rb_node *old, struct rb_node *new) {}
static inline void dummy_rotate(struct rb_node *old, struct rb_node *new) {}

static const struct rb_augment_callbacks dummy_callbacks = {
    .propagate = dummy_propagate,
    .copy = dummy_copy,
    .rotate = dummy_rotate
};
```

### Node Replacement Optimization

Fast node replacement without full delete/insert:

```c
void rb_replace_node(struct rb_node *victim, struct rb_node *new,
                     struct rb_root *root)
{
    struct rb_node *parent = rb_parent(victim);
    
    /* Copy the pointers/colour from the victim to the replacement */
    *new = *victim;
    
    /* Set the surrounding nodes to point to the replacement */
    if (victim->rb_left)
        rb_set_parent(victim->rb_left, new);
    if (victim->rb_right)
        rb_set_parent(victim->rb_right, new);
    __rb_change_child(victim, new, parent, root);
}
```

## Examples and Best Practices

### Basic Usage Example

```c
struct my_node {
    int key;
    char data[64];
    struct rb_node node;
};

static struct rb_root my_tree = RB_ROOT;

static bool my_less(struct rb_node *a, const struct rb_node *b)
{
    struct my_node *node_a = rb_entry(a, struct my_node, node);
    struct my_node *node_b = rb_entry(b, struct my_node, node);
    return node_a->key < node_b->key;
}

static int my_cmp(const void *key, const struct rb_node *node)
{
    int search_key = *(const int *)key;
    struct my_node *my_node = rb_entry(node, struct my_node, node);
    
    if (search_key < my_node->key)
        return -1;
    else if (search_key > my_node->key)
        return 1;
    else
        return 0;
}

// Insertion
void insert_node(struct my_node *new_node)
{
    rb_add(&new_node->node, &my_tree, my_less);
}

// Lookup
struct my_node *find_node(int key)
{
    struct rb_node *node = rb_find(&key, &my_tree, my_cmp);
    return node ? rb_entry(node, struct my_node, node) : NULL;
}

// Deletion
void delete_node(struct my_node *node)
{
    rb_erase(&node->node, &my_tree);
    RB_CLEAR_NODE(&node->node);  // Mark as deleted
}
```

### Augmented Tree Example

```c
struct interval_node {
    unsigned long start;
    unsigned long end;
    unsigned long max_end;      // Augmented data
    struct rb_node rb;
};

static bool interval_less(struct rb_node *a, const struct rb_node *b)
{
    struct interval_node *ia = rb_entry(a, struct interval_node, rb);
    struct interval_node *ib = rb_entry(b, struct interval_node, rb);
    return ia->start < ib->start;
}

static bool interval_compute_max(struct interval_node *node, bool exit)
{
    unsigned long max = node->end;
    
    if (node->rb.rb_left) {
        struct interval_node *left = rb_entry(node->rb.rb_left, 
                                             struct interval_node, rb);
        if (left->max_end > max)
            max = left->max_end;
    }
    
    if (node->rb.rb_right) {
        struct interval_node *right = rb_entry(node->rb.rb_right,
                                              struct interval_node, rb);
        if (right->max_end > max)
            max = right->max_end;
    }
    
    if (exit && node->max_end == max)
        return true;
    
    node->max_end = max;
    return false;
}

RB_DECLARE_CALLBACKS(static, interval_callbacks, struct interval_node, rb,
                      unsigned long, max_end, interval_compute_max)
```

### Best Practices

1. **Always use `rb_entry_safe()` for potentially NULL nodes**
2. **Mark deleted nodes with `RB_CLEAR_NODE()`**
3. **Use cached trees when frequent minimum access is needed**
4. **Prefer the modern `rb_add()` API over manual linking**
5. **Use `rbtree_postorder_for_each_entry_safe()` for destruction**
6. **Consider augmented trees for range queries**
7. **Use `__always_inline` helpers for performance-critical paths**
8. **Implement proper comparison functions (stable ordering)**

### Performance Considerations

- **Tree Height**: Worst case 2⌊log₂(n+1)⌋, typical case ≈ log₂(n)
- **Cache Performance**: Excellent for traversal, moderate for random access
- **Memory Overhead**: 24 bytes per node (64-bit), 12 bytes (32-bit)
- **Lockless Lookups**: Supported but may miss concurrent modifications
- **RCU Integration**: Available through latched trees and RCU variants

The Linux kernel's red-black tree implementation represents a mature, highly optimized data structure suitable for a wide range of kernel subsystems requiring ordered data with guaranteed logarithmic performance bounds.