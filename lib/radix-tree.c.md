# radix-tree.c - Radix Tree Data Structure Implementation

## Overview

The `radix-tree.c` file implements a compressed radix tree (also known as a trie) data structure optimized for storing and retrieving data with integer keys. This is a fundamental data structure used throughout the Linux kernel for efficient sparse array implementations, particularly in memory management, file systems, and ID allocation.

## File Location
- **Path**: `lib/radix-tree.c`
- **License**: GPL-2.0-or-later
- **Authors**: 
  - Momchil Velikov (2001)
  - Christoph Hellwig (2001)
  - Christoph Lameter, SGI (2005)
  - Nick Piggin (2006)
  - Konstantin Khlebnikov (2012)
  - Matthew Wilcox, Intel (2016)
  - Ross Zwisler, Intel (2016)

## Core Concepts

### Radix Tree Structure
A radix tree is a space-efficient trie where nodes with single children are merged with their parents. Key characteristics:
- **Variable Height**: Tree grows as needed based on key space
- **Compressed**: Nodes with single children are compressed
- **Tagged**: Supports multiple tag bits per entry for fast searching
- **RCU Safe**: Supports lockless readers via RCU

### Key Parameters
- **RADIX_TREE_MAP_SHIFT**: Bits per level (typically 6, giving 64 children per node)
- **RADIX_TREE_MAP_SIZE**: Children per node (64)
- **RADIX_TREE_MAP_MASK**: Mask for extracting offset (0x3f)
- **RADIX_TREE_MAX_PATH**: Maximum tree height

## Memory Management

### Node Allocation
```c
struct kmem_cache *radix_tree_node_cachep;
```
Dedicated SLAB cache for radix tree nodes, providing efficient allocation and deallocation.

### Preloading System
```c
DEFINE_PER_CPU(struct radix_tree_preload, radix_tree_preloads) = {
    .lock = INIT_LOCAL_LOCK(lock),
};
```

**Purpose**: Avoid memory allocation in atomic contexts
**Size Calculation**: 
- **RADIX_TREE_PRELOAD_SIZE**: (RADIX_TREE_MAX_PATH * 2 - 1)
- **IDR_PRELOAD_SIZE**: (IDR_MAX_PATH * 2 - 1)

**Worst Case**: Inserting at index ULONG_MAX in empty tree requires maximum preload.

## Core Data Structure Operations

### Node Pointer Encoding

#### `entry_to_node(void *ptr)`
Extracts node pointer from encoded entry.
- **Parameters**: Encoded pointer
- **Returns**: Raw node pointer
- **Implementation**: Masks off `RADIX_TREE_INTERNAL_NODE` bit

#### `node_to_entry(void *ptr)`
Encodes node pointer for storage in parent.
- **Parameters**: Raw node pointer
- **Returns**: Encoded pointer with internal node bit set
- **Purpose**: Distinguishes internal nodes from leaf data

### Tree Navigation

#### `get_slot_offset(const struct radix_tree_node *parent, void __rcu **slot)`
Calculates slot offset within parent node.
- **Parameters**: Parent node and slot pointer
- **Returns**: Offset index (0 for root)
- **Usage**: Determines position within parent's slot array

#### `radix_tree_descend(const struct radix_tree_node *parent, struct radix_tree_node **nodep, unsigned long index)`
Descends one level in the tree following the path for given index.
- **Parameters**: Parent node, output node pointer, target index
- **Returns**: Slot offset in parent
- **Logic**:
  1. Calculates offset using parent's shift value
  2. Dereferences slot using RCU protection
  3. Returns both the offset and next node

### Memory Context

#### `root_gfp_mask(const struct radix_tree_root *root)`
Extracts GFP allocation flags from root structure.
- **Parameters**: Tree root
- **Returns**: GFP flags for memory allocation
- **Masking**: Filters valid GFP bits excluding zone mask

## Tagging System

### Tag Operations

#### `tag_set(struct radix_tree_node *node, unsigned int tag, int offset)`
Sets a tag bit for specific slot in node.
- **Parameters**: Node, tag number, slot offset
- **Implementation**: Uses `__set_bit()` for atomic operation

#### `tag_clear(struct radix_tree_node *node, unsigned int tag, int offset)`
Clears a tag bit for specific slot in node.
- **Parameters**: Node, tag number, slot offset
- **Implementation**: Uses `__clear_bit()` for atomic operation

#### `tag_get(const struct radix_tree_node *node, unsigned int tag, int offset)`
Tests if tag bit is set for specific slot.
- **Parameters**: Node, tag number, slot offset
- **Returns**: Non-zero if tag is set
- **Implementation**: Uses `test_bit()` for atomic read

### Root Tag Management

#### `root_tag_set(struct radix_tree_root *root, unsigned tag)`
Sets tag bit in root flags to indicate tagged entries exist.
- **Parameters**: Root structure, tag number
- **Implementation**: Sets bit at position (tag + ROOT_TAG_SHIFT)

#### `root_tag_clear(struct radix_tree_root *root, unsigned tag)`
Clears tag bit in root flags when no tagged entries remain.
- **Parameters**: Root structure, tag number
- **Implementation**: Clears bit at position (tag + ROOT_TAG_SHIFT)

#### `root_tag_clear_all(struct radix_tree_root *root)`
Clears all tag bits in root flags.
- **Parameters**: Root structure
- **Purpose**: Used during tree destruction or reset

#### `root_tag_get(const struct radix_tree_root *root, unsigned tag)`
Tests if any entries with given tag exist in tree.
- **Parameters**: Root structure, tag number
- **Returns**: Non-zero if tagged entries exist
- **Optimization**: Allows early exit from searches

#### `root_tags_get(const struct radix_tree_root *root)`
Retrieves all root tag bits as combined value.
- **Parameters**: Root structure
- **Returns**: Tag bits shifted to lower positions
- **Usage**: Bulk tag operations

### Tag Utility Functions

#### `any_tag_set(const struct radix_tree_node *node, unsigned int tag)`
Checks if any slot in node has specified tag set.
- **Parameters**: Node, tag number
- **Returns**: 1 if any slot tagged, 0 otherwise
- **Implementation**: Iterates through tag bitmap longs

#### `all_tag_set(struct radix_tree_node *node, unsigned int tag)`
Sets specified tag on all slots in node.
- **Parameters**: Node, tag number
- **Implementation**: Uses `bitmap_fill()` for efficiency

### Advanced Tag Search

#### `radix_tree_find_next_bit(struct radix_tree_node *node, unsigned int tag, unsigned long offset)`
Optimized function to find next set tag bit starting from offset.
- **Parameters**: Node, tag number, starting offset
- **Returns**: Next set bit offset or RADIX_TREE_MAP_SIZE if none found
- **Optimization**: 
  - Unrolled for constant-size arrays
  - Word-at-a-time processing
  - Uses `__ffs()` for efficient bit finding

**Algorithm**:
1. **Partial Word**: Handle remaining bits in first word
2. **Word Alignment**: Advance to next word boundary
3. **Full Words**: Process complete words until match found
4. **Bit Extraction**: Use `__ffs()` to find exact bit position

## IDR Integration

### IDR Differences
```c
#define IDR_INDEX_BITS    (8 * sizeof(int) - 1)  // Signed integer space
#define IDR_MAX_PATH      (DIV_ROUND_UP(IDR_INDEX_BITS, RADIX_TREE_MAP_SHIFT))
#define IDR_PRELOAD_SIZE  (IDR_MAX_PATH * 2 - 1)
```

#### `is_idr(const struct radix_tree_root *root)`
Determines if tree is used as an IDR (ID allocator).
- **Parameters**: Tree root
- **Returns**: True if ROOT_IS_IDR flag set
- **Purpose**: IDRs have different semantics and constraints

## Performance Optimizations

### Per-CPU Preloading
- **Lock-Free Allocation**: Pre-allocated nodes avoid atomic context issues
- **Per-CPU Storage**: Reduces cache line contention
- **Local Locking**: Fine-grained locking per CPU

### RCU Integration
- **Lockless Reads**: Readers don't need locks in most cases
- **Safe Traversal**: RCU ensures nodes remain valid during traversal
- **Memory Barriers**: Proper ordering for concurrent access

### Cache Efficiency
- **SLAB Cache**: Dedicated cache improves allocation performance
- **Node Layout**: Optimized for common access patterns
- **Prefetching**: Strategic prefetching in hot paths

## Tree Modification Operations

### Height Extension
Trees automatically grow when inserting beyond current capacity:
1. **Capacity Check**: Determine if current height sufficient
2. **New Root**: Allocate new root node if extension needed
3. **Height Update**: Adjust shift values for new height
4. **Tag Propagation**: Maintain tag consistency through height changes

### Node Compression
Empty intermediate nodes are removed to maintain efficiency:
1. **Empty Detection**: Identify nodes with no children
2. **Parent Update**: Modify parent to skip empty node
3. **RCU Cleanup**: Use RCU to safely free empty nodes
4. **Tag Cleanup**: Clear tags in parent when removing nodes

## Error Handling and Edge Cases

### Memory Allocation Failures
- **Preload Exhaustion**: Handle preload pool depletion gracefully
- **Atomic Context**: Use preloaded nodes when allocation impossible
- **Recovery**: Partial operations can be rolled back cleanly

### Concurrent Access
- **RCU Protection**: Readers protected from writers
- **Lock Ordering**: Consistent lock ordering prevents deadlocks
- **Tag Consistency**: Tags remain consistent during concurrent modifications

### Index Range Validation
- **Bounds Checking**: Validate indices against maximum tree capacity
- **Signed/Unsigned**: Proper handling of signed IDR vs unsigned radix tree indices
- **Overflow Protection**: Prevent integer overflow in calculations

## Integration with XArray

### Shared Infrastructure
Modern kernels use XArray as the primary interface, with radix trees as the underlying implementation:
- **RADIX_TREE_RETRY**: Maps to XA_RETRY_ENTRY for restart semantics
- **Compatibility Layer**: Maintains radix tree API for existing code
- **Gradual Migration": Existing code gradually migrated to XArray API

## Debugging and Validation

### Consistency Checks
- **Tag Propagation**: Verify tags properly propagated to root
- **Height Consistency**: Ensure tree height matches actual structure
- **Node Linkage**: Validate parent-child relationships

### Memory Leak Detection
- **kmemleak Integration**: Proper annotation for memory leak detection
- **Reference Counting**: Track node lifecycle for leak prevention
- **RCU Validation**: Ensure proper RCU grace period handling

This radix tree implementation provides a highly optimized, scalable data structure that serves as the foundation for many kernel subsystems requiring efficient sparse array operations with tagging support and concurrent access capabilities.