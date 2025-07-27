# lib/idr.c - ID Allocator Implementation

## Overview

This file implements the IDR (ID Allocator) and IDA (ID Allocator Array) data structures for the Linux kernel. These are efficient mechanisms for allocating and managing unique integer identifiers that can be associated with kernel objects.

## Core Data Structures

### IDR (ID Allocator)
- **Purpose**: Associates unique integer IDs with pointer values
- **Implementation**: Built on top of radix trees (XArray)
- **Use Case**: When you need to map IDs to specific objects/pointers

### IDA (ID Allocator Array)  
- **Purpose**: Allocates unique integer IDs without associated data
- **Implementation**: More memory-efficient than IDR, uses bitmaps
- **Use Case**: When you only need unique IDs without pointer associations

## Key Functions

### IDR Functions

#### `idr_alloc_u32()`
```c
int idr_alloc_u32(struct idr *idr, void *ptr, u32 *nextid, unsigned long max, gfp_t gfp)
```
- **Purpose**: Allocates an ID in the range [nextid, max] and associates it with ptr
- **Parameters**:
  - `idr`: IDR handle
  - `ptr`: Pointer to associate with the new ID
  - `nextid`: Pointer to desired starting ID (updated with allocated ID)
  - `max`: Maximum ID (inclusive)
  - `gfp`: Memory allocation flags
- **Returns**: 0 on success, -ENOMEM/-ENOSPC on failure
- **Thread Safety**: Requires external locking for modifications

#### `idr_alloc()`
```c
int idr_alloc(struct idr *idr, void *ptr, int start, int end, gfp_t gfp)
```
- **Purpose**: Wrapper around idr_alloc_u32() with signed integer interface
- **Range**: [start, end) - note end is exclusive
- **Returns**: Allocated ID on success, negative error code on failure

#### `idr_alloc_cyclic()`
```c
int idr_alloc_cyclic(struct idr *idr, void *ptr, int start, int end, gfp_t gfp)
```
- **Purpose**: Allocates IDs cyclically starting from the last allocated ID
- **Behavior**: Wraps around to start if no free IDs found before end
- **Use Case**: Distributing IDs evenly across the range

#### `idr_remove()`
```c
void *idr_remove(struct idr *idr, unsigned long id)
```
- **Purpose**: Removes an ID from the IDR and returns associated pointer
- **Returns**: Previously associated pointer or NULL if ID not found
- **Thread Safety**: Requires external locking

#### `idr_find()`
```c
void *idr_find(const struct idr *idr, unsigned long id)
```
- **Purpose**: Looks up pointer associated with an ID
- **Thread Safety**: Can be called under RCU read lock
- **Returns**: Associated pointer or NULL

#### `idr_for_each()`
```c
int idr_for_each(const struct idr *idr, int (*fn)(int id, void *p, void *data), void *data)
```
- **Purpose**: Iterates through all stored pointers
- **Callback**: Called for each entry with (ID, pointer, user_data)
- **Thread Safety**: Can be called concurrently with alloc/remove under RCU

#### `idr_replace()`
```c
void *idr_replace(struct idr *idr, void *ptr, unsigned long id)
```
- **Purpose**: Replaces the pointer for an existing ID
- **Returns**: Old pointer value or error code
- **Thread Safety**: Can be called under RCU with proper synchronization

### IDA Functions

#### `ida_alloc_range()`
```c
int ida_alloc_range(struct ida *ida, unsigned int min, unsigned int max, gfp_t gfp)
```
- **Purpose**: Core IDA allocation function
- **Implementation Details**:
  - Uses XArray with bitmaps for storage efficiency
  - Small allocations use value entries (embedded in XArray slots)
  - Larger allocations use 128-byte bitmap structures
  - Automatic conversion between value entries and bitmaps
- **Thread Safety**: Handles its own locking internally
- **Returns**: Allocated ID or negative error code

#### `ida_find_first_range()`
```c
int ida_find_first_range(struct ida *ida, unsigned int min, unsigned int max)
```
- **Purpose**: Finds the lowest allocated ID in the given range
- **Use Case**: Scanning for allocated IDs
- **Returns**: First allocated ID or -ENOENT if none found

#### `ida_free()`
```c
void ida_free(struct ida *ida, unsigned int id)
```
- **Purpose**: Releases an allocated ID
- **Implementation**: Clears the bit in the bitmap/value entry
- **Error Handling**: WARN()s if freeing unallocated ID
- **Thread Safety**: Handles its own locking

#### `ida_destroy()`
```c
void ida_destroy(struct ida *ida)
```
- **Purpose**: Frees all IDs and releases all resources
- **Use Case**: Cleanup when IDA is no longer needed
- **Memory Management**: Frees all bitmap structures

## Implementation Details

### IDR Implementation
- **Radix Tree Base**: Built on the XArray radix tree implementation
- **Base Offset**: Supports custom base values via `idr_base` field
- **Free Tracking**: Uses IDR_FREE tag to track available slots
- **Memory Barriers**: Proper memory ordering in radix_tree_iter_replace()

### IDA Implementation
- **Bitmap Strategy**: Uses two-tier approach for memory efficiency:
  1. **Value Entries**: For sparse allocations, bits stored directly in XArray slots
  2. **Bitmap Entries**: 128-byte bitmaps for dense allocations
- **Free Marking**: XA_FREE_MARK tracks entries with available bits
- **Optimization**: Automatic conversion between value and bitmap entries
- **Lock Management**: Internal spinlock for thread safety

### Memory Efficiency
- **IDA vs IDR**: IDA uses ~1 bit per ID vs ~1 pointer per ID for IDR
- **Sparse Handling**: Value entries avoid bitmap allocation for small sets
- **Dense Handling**: Full bitmaps for areas with many allocated IDs

## Locking and Thread Safety

### IDR Locking
- **External Locking Required**: Callers must provide synchronization for modifications
- **RCU Support**: Read operations can use RCU read lock
- **Concurrent Operations**: Safe read-only access during modifications

### IDA Locking  
- **Internal Locking**: IDA handles all locking internally
- **IRQ Safe**: Uses irq-safe spinlocks (xas_lock_irqsave)
- **No External Sync Needed**: Completely thread-safe interface

## Error Handling

### Common Error Codes
- **-ENOMEM**: Memory allocation failure
- **-ENOSPC**: No free IDs available in range  
- **-ENOENT**: ID not found (for lookups)
- **-EINVAL**: Invalid parameters

### Debugging Support
- **WARN_ON_ONCE**: Detects invalid usage patterns
- **Debug Dumps**: Non-kernel debug functions for IDA structure inspection

## Performance Characteristics

### IDR Performance
- **Lookup**: O(log n) radix tree lookup
- **Allocation**: O(log n) with potential tree expansion
- **Iteration**: O(n) for all entries

### IDA Performance  
- **Lookup**: O(log n) for checking allocation status
- **Allocation**: O(log n) with bitmap search within slots
- **Memory**: Much more efficient than IDR for ID-only use cases

## Usage Examples

### IDR Usage Pattern
```c
struct idr my_idr;
idr_init(&my_idr);

// Allocate ID for object
int id = idr_alloc(&my_idr, my_object, 0, 0, GFP_KERNEL);

// Look up object
struct my_object *obj = idr_find(&my_idr, id);

// Remove and get object
obj = idr_remove(&my_idr, id);

idr_destroy(&my_idr);
```

### IDA Usage Pattern  
```c
struct ida my_ida;
ida_init(&my_ida);

// Allocate ID
int id = ida_alloc(&my_ida, GFP_KERNEL);

// Free ID
ida_free(&my_ida, id);

ida_destroy(&my_ida);
```

## Security Considerations

- **ID Exhaustion**: Applications should handle -ENOSPC gracefully
- **Double Free Detection**: IDA warns on double-free attempts
- **Integer Overflow**: Proper bounds checking prevents overflow
- **RCU Safety**: Proper RCU usage prevents use-after-free

## Integration Points

### XArray Integration
- **IDR**: Uses XArray as underlying radix tree implementation
- **IDA**: Uses XArray for bitmap storage and indexing
- **Marks**: Leverages XArray mark system for free space tracking

### Memory Management
- **GFP Flags**: Respects caller's memory allocation preferences
- **NOWAIT Allocation**: Uses GFP_NOWAIT for atomic contexts
- **Memory Reclaim**: Proper cleanup in error paths

This implementation provides efficient, scalable ID allocation suitable for kernel subsystems requiring unique identifier management with optional pointer associations.