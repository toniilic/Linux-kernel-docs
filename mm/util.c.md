# mm/util.c - Memory Management Utility Functions

## Overview

The `util.c` file in the memory management subsystem provides fundamental utility functions for memory allocation, string duplication, and other memory-related operations. These functions serve as building blocks for higher-level kernel functionality and provide safe, efficient alternatives to standard C library functions.

## File Location
- **Path**: `mm/util.c`
- **License**: GPL-2.0-only
- **Purpose**: Core memory management utility functions

## Core Memory Utility Functions

### Conditional Memory Management

#### `kfree_const(const void *x)`
Conditionally frees memory only if it's not in the kernel's read-only data section.

**Parameters**: 
- `x`: Pointer to memory that may or may not need freeing

**Behavior**:
- **Read-Only Data**: Does nothing if pointer is in .rodata section
- **Dynamic Memory**: Calls `kfree()` if memory was dynamically allocated
- **Safety**: Prevents attempts to free const/static data

**Use Case**: Safe cleanup when unsure if data is static or allocated

**Implementation**:
```c
void kfree_const(const void *x)
{
    if (!is_kernel_rodata((unsigned long)x))
        kfree(x);
}
```

### String Duplication Functions

#### `__kmemdup_nul(const char *s, size_t len, gfp_t gfp)`
Internal helper that creates a NUL-terminated copy of potentially unterminated data.

**Parameters**:
- `s`: Source data to copy
- `len`: Length of data (excluding NUL terminator)
- `gfp`: GFP allocation flags

**Returns**: Newly allocated NUL-terminated string or NULL on failure

**Key Features**:
- **Always Inline**: Marked `__always_inline` for performance
- **NUL Termination**: Guarantees NUL termination regardless of source
- **Track Caller**: Uses `kmalloc_track_caller()` for debug info
- **Size Safety**: Allocates len + 1 bytes for terminator

**Implementation**:
```c
buf = kmalloc_track_caller(len + 1, gfp);
if (!buf)
    return NULL;
memcpy(buf, s, len);
buf[len] = '\0';  /* Force NUL termination */
```

#### `kstrdup(const char *s, gfp_t gfp)`
Duplicates a NUL-terminated string.

**Parameters**:
- `s`: Source string to duplicate
- `gfp`: GFP allocation flags

**Returns**: Newly allocated copy of string or NULL

**Behavior**:
- **NULL Handling**: Returns NULL if source is NULL
- **Length Calculation**: Uses `strlen()` to determine length
- **Memory Efficient**: Only allocates needed space

**Use Cases**: Standard string duplication for kernel code

#### `kstrdup_const(const char *s, gfp_t gfp)`
Conditionally duplicates a string, returning original if it's in read-only data.

**Parameters**:
- `s`: Source string
- `gfp`: GFP allocation flags

**Returns**: Original string if in .rodata, otherwise newly allocated copy

**Optimization**: Avoids unnecessary allocation for const strings

**Important Notes**:
- **Must use kfree_const()**: Result must be freed with `kfree_const()`
- **No krealloc()**: Result cannot be passed to `krealloc()`
- **Memory Efficiency**: Saves memory for frequently used constant strings

#### `kstrndup(const char *s, size_t max, gfp_t gfp)`
Duplicates at most `max` characters from a string.

**Parameters**:
- `s`: Source string
- `max`: Maximum characters to copy
- `gfp`: GFP allocation flags

**Returns**: Newly allocated string copy or NULL

**Features**:
- **Length Limiting**: Copies at most `max` characters
- **NUL Termination**: Always NUL-terminated
- **Safe**: Uses `strnlen()` to avoid buffer overruns

**Note**: Documentation suggests `kmemdup_nul()` if exact size is known

### Binary Data Duplication

#### `kmemdup_noprof(const void *src, size_t len, gfp_t gfp)`
Duplicates arbitrary memory region.

**Parameters**:
- `src`: Source memory region
- `len`: Number of bytes to copy
- `gfp`: GFP allocation flags

**Returns**: Newly allocated copy or NULL on failure

**Characteristics**:
- **Physically Contiguous**: Result is guaranteed contiguous
- **Exact Copy**: Byte-for-byte copy of source
- **No Profiling**: "_noprof" version doesn't include memory profiling
- **Track Caller**: Uses caller tracking for debugging

#### `kmemdup_array(const void *src, size_t count, size_t element_size, gfp_t gfp)`
Duplicates an array with overflow protection.

**Parameters**:
- `src`: Source array
- `count`: Number of elements
- `element_size`: Size of each element
- `gfp`: GFP allocation flags

**Returns**: Newly allocated array copy or NULL

**Safety Features**:
- **Overflow Protection**: Uses `size_mul()` to detect integer overflow
- **Type Safety**: Separates count and element size for clarity
- **Standard Interface**: Consistent with other array functions

#### `kvmemdup(const void *src, size_t len, gfp_t gfp)`
Duplicates memory region using vmalloc for large allocations.

**Parameters**:
- `src`: Source memory region
- `len`: Number of bytes to copy
- `gfp`: GFP allocation flags

**Returns**: Newly allocated copy or NULL on failure

**Key Differences from kmemdup**:
- **May Be Non-Contiguous**: Result may use vmalloc (non-contiguous)
- **Large Allocations**: Better for large memory regions
- **Must Use kvfree()**: Result must be freed with `kvfree()`
- **Fallback Strategy**: Uses kmalloc first, vmalloc for large sizes

#### `kmemdup_nul(const char *s, size_t len, gfp_t gfp)`
Creates NUL-terminated string from unterminated data.

**Parameters**:
- `s`: Source data (may be unterminated)
- `len`: Length of source data
- `gfp`: GFP allocation flags

**Returns**: NUL-terminated string or NULL

**Use Cases**:
- **Protocol Parsing**: Converting network data to strings
- **Binary Data**: Creating strings from binary formats
- **Safe String Handling**: When source termination is uncertain

## Memory Bucket System

### User Space Allocation Buckets
```c
static kmem_buckets *user_buckets __ro_after_init;

static int __init init_user_buckets(void)
```

**Purpose**: Specialized memory allocation buckets for user-space-influenced allocations

**Security Feature**: Helps with memory safety by segregating user-influenced allocations

**Initialization**: Set up during kernel initialization (`__init`)

**Read-Only**: Marked `__ro_after_init` to prevent modification after boot

## Memory Safety Features

### Read-Only Data Detection
- **is_kernel_rodata()**: Checks if pointer is in kernel's read-only section
- **Prevents Double-Free**: Avoids freeing static/const data
- **Memory Layout Aware**: Uses kernel memory layout information

### Overflow Protection
- **size_mul()**: Safe multiplication with overflow detection
- **Bounds Checking**: Length validation in string functions
- **NULL Handling**: Consistent NULL pointer handling

### Caller Tracking
- **kmalloc_track_caller()**: Preserves allocation source information
- **Debug Support**: Helps identify memory leak sources
- **Stack Unwinding**: Maintains call stack for debugging

## Performance Considerations

### Optimization Strategies
- **Inline Functions**: Critical paths marked for inlining
- **Const Optimization**: Avoids unnecessary duplication of const data
- **Contiguous vs Non-Contiguous**: Choice based on size requirements

### Memory Efficiency
- **Exact Sizing**: Allocates only necessary memory
- **Bucket System**: Specialized allocators for different use patterns
- **Fallback Strategies**: Multiple allocation strategies for different sizes

## Integration Points

### SLAB/SLUB Integration
- **Allocation Functions**: Uses kernel's primary memory allocators
- **GFP Flags**: Full support for all GFP allocation modes
- **NUMA Awareness**: Supports NUMA-aware allocations

### Security Framework
- **Bucket Isolation**: Separates user-influenced allocations
- **Const Data Protection**: Prevents modification of read-only data
- **Safe String Handling**: Provides safe alternatives to unsafe C functions

### Debugging Infrastructure
- **Caller Tracking**: Maintains allocation source information
- **KUnit Testing**: Includes test infrastructure support
- **Memory Profiling**: Integration with kernel memory profiling

## Error Handling

### Allocation Failures
- **Consistent NULL Returns**: All functions return NULL on allocation failure
- **No Partial State**: Functions either succeed completely or fail cleanly
- **GFP Flag Respect**: Honors allocation constraints (atomic, etc.)

### Input Validation
- **NULL Source Handling**: Safe handling of NULL input pointers
- **Size Validation**: Prevents integer overflow in size calculations
- **Memory Bounds**: Respects maximum allocation sizes

## Export Status

All major functions are exported with `EXPORT_SYMBOL()` or `EXPORT_SYMBOL_GPL()`, making them available to:
- **Kernel Modules**: Loadable kernel modules can use these functions
- **Driver Code**: Device drivers have access to these utilities
- **Subsystem Code**: Other kernel subsystems can build on these primitives

This utility file provides the foundation for safe, efficient memory management throughout the Linux kernel, offering alternatives to potentially unsafe C library functions while integrating with kernel-specific features like GFP flags, NUMA awareness, and security frameworks.