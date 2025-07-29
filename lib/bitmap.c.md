# bitmap.c - Bitmap Operations Implementation

## Overview

The `bitmap.c` file provides comprehensive implementation of bitmap operations for the Linux kernel. Bitmaps are fundamental data structures used throughout the kernel for efficient representation and manipulation of sets of bits, commonly used for resource allocation, CPU masks, IRQ management, and various other purposes.

## File Location
- **Path**: `lib/bitmap.c`
- **License**: GPL-2.0-only
- **Purpose**: Helper functions for bitmap.h operations

## Core Concepts

### Bitmap Structure
Bitmaps in Linux are implemented as arrays of `unsigned long` values:
- **Word-Based**: Uses machine word size for efficiency
- **Variable Length**: Supports arbitrary bit counts (not just multiples of BITS_PER_LONG)
- **Unused Bits**: Trailing bits in final word are "don't care" 
- **Little-Endian Friendly**: More natural on little-endian architectures

### Key Design Principles
- **Efficient Implementation**: Word-level operations where possible
- **Unused Bit Handling**: Careful masking to ignore unused bits in results
- **Endian Awareness**: Consistent behavior across architectures
- **Safety**: Proper bounds checking and bit masking

## Core Bitmap Functions

### Comparison Operations

#### `__bitmap_equal(const unsigned long *bitmap1, const unsigned long *bitmap2, unsigned int bits)`
Compares two bitmaps for equality.

**Parameters**:
- `bitmap1`: First bitmap to compare
- `bitmap2`: Second bitmap to compare  
- `bits`: Number of valid bits to compare

**Returns**: `true` if bitmaps are equal, `false` otherwise

**Algorithm**:
1. **Word-by-Word Compare**: Compare complete words using fast integer comparison
2. **Partial Word Handling**: For remaining bits, use `BITMAP_LAST_WORD_MASK()` to ignore unused bits
3. **XOR Test**: Use XOR to detect differences in final partial word

**Implementation Details**:
```c
unsigned int k, lim = bits/BITS_PER_LONG;
for (k = 0; k < lim; ++k)
    if (bitmap1[k] != bitmap2[k])
        return false;
        
if (bits % BITS_PER_LONG)
    if ((bitmap1[k] ^ bitmap2[k]) & BITMAP_LAST_WORD_MASK(bits))
        return false;
```

#### `__bitmap_or_equal(const unsigned long *bitmap1, const unsigned long *bitmap2, const unsigned long *bitmap3, unsigned int bits)`
Tests if `(bitmap1 | bitmap2) == bitmap3`.

**Parameters**:
- `bitmap1`: First source bitmap
- `bitmap2`: Second source bitmap  
- `bitmap3`: Expected result bitmap
- `bits`: Number of valid bits

**Returns**: `true` if OR operation equals bitmap3

**Purpose**: Efficient three-way comparison avoiding temporary bitmap allocation

**Optimization**: Performs OR and comparison in single pass

### Bitwise Operations

#### `__bitmap_complement(unsigned long *dst, const unsigned long *src, unsigned int bits)`
Computes bitwise complement (NOT operation).

**Parameters**:
- `dst`: Destination bitmap
- `src`: Source bitmap
- `bits`: Number of valid bits

**Operation**: Sets `dst[i] = ~src[i]` for all words

**Word-Level Efficiency**: Operates on complete words using native NOT operation

**Note**: Unused bits in final word are complemented but don't affect semantic results

### Shift Operations

#### `__bitmap_shift_right(unsigned long *dst, const unsigned long *src, unsigned shift, unsigned nbits)`
Performs logical right shift on bitmap.

**Parameters**:
- `dst`: Destination bitmap
- `src`: Source bitmap
- `shift`: Number of bit positions to shift right
- `nbits`: Total number of valid bits

**Behavior**:
- **Logical Shift**: Zeros are shifted in from the left (MS positions)
- **Lost Bits**: Bits shifted off the right are discarded
- **Direction**: MS (Most Significant) → LS (Least Significant)

**Algorithm Components**:
1. **Word Offset**: `off = shift/BITS_PER_LONG` - complete word shifts
2. **Bit Remainder**: `rem = shift % BITS_PER_LONG` - sub-word bit shifts
3. **Cross-Word Handling**: Combines bits from adjacent words when needed

**Implementation Strategy**:
```c
unsigned off = shift/BITS_PER_LONG, rem = shift % BITS_PER_LONG;
for (k = 0; off + k < lim; ++k) {
    unsigned long upper, lower;
    // Complex bit manipulation to handle cross-word shifts
}
```

## Bit Manipulation Utilities

### Masking Operations
- **BITMAP_LAST_WORD_MASK(bits)**: Creates mask for valid bits in final word
- **BITS_TO_LONGS(bits)**: Calculates number of words needed for bit count
- **BITS_PER_LONG**: Architecture-specific word size in bits

### Endianness Considerations
The documentation notes that bitmap ordering is more natural on little-endian architectures:
- **Bit Numbering**: Consistent bit numbering across architectures
- **Word Boundaries**: Proper handling of word-boundary crossings
- **Architecture Headers**: Special handling in big-endian architectures (PowerPC, S390)

## Performance Characteristics

### Word-Level Operations
- **Bulk Processing**: Processes multiple bits per operation using word-size operations
- **Loop Unrolling**: Efficient loops over word arrays
- **Branch Prediction**: Minimal branching in hot paths

### Memory Access Patterns
- **Sequential Access**: Linear memory access patterns for cache efficiency
- **Word Alignment**: Takes advantage of word-aligned memory access
- **Minimal Overhead**: Direct bit manipulation without excessive function calls

## Usage Patterns

### Common Use Cases
- **Resource Allocation**: Track allocated/free resources (memory pages, IRQs)
- **CPU Masks**: Represent sets of CPUs for scheduling and affinity
- **Hardware State**: Track hardware resource states
- **Set Operations**: Efficient set membership and operations

### Integration Points
- **CPU Management**: cpumask operations built on bitmap functions
- **Memory Management**: Page allocation bitmaps
- **IRQ Subsystem**: IRQ affinity and routing
- **Device Drivers**: Hardware resource tracking

## Error Handling and Safety

### Bounds Checking
- **Bit Count Validation**: Functions respect specified bit counts
- **Word Boundary Safety**: Proper handling of partial final words
- **Unused Bit Masking**: Ensures unused bits don't affect results

### Memory Safety
- **No Buffer Overruns**: Operations stay within specified bitmap bounds
- **Consistent State**: Operations leave bitmaps in consistent state
- **Atomic Word Operations**: Word-level operations are naturally atomic on most architectures

## Export Status

Functions marked with `EXPORT_SYMBOL()` are available to kernel modules:
- **Module Access**: Kernel modules can use bitmap operations
- **Driver Support**: Device drivers have access to bitmap functionality
- **Subsystem Integration**: Other kernel subsystems can build on bitmap operations

## Documentation Integration

### DOC Comments
The file includes comprehensive DOC comments explaining:
- **Bitmap Structure**: How bitmaps are represented in memory
- **Bit Ordering**: Architecture-specific considerations
- **Unused Bit Handling**: How trailing bits are managed
- **Performance Notes**: Efficiency characteristics

### Function Documentation
Each function includes detailed parameter descriptions and behavioral notes:
- **Parameter Types**: Clear typing and const-correctness
- **Return Values**: Explicit return value semantics
- **Side Effects**: Documentation of what gets modified

## Testing and Validation

### Bit Mask Validation
- **Last Word Masking**: Proper masking ensures unused bits don't affect results
- **Edge Case Handling**: Correct behavior for edge cases (empty bitmaps, single bits)
- **Cross-Architecture**: Consistent behavior across different architectures

### Performance Testing
- **Benchmarking**: Operations optimized for common use patterns
- **Scalability**: Efficient performance with large bitmaps
- **Memory Efficiency**: Minimal memory overhead and fragmentation

This bitmap implementation provides the foundational bit manipulation capabilities that many other kernel subsystems depend on, offering both performance and correctness for critical system operations involving bit sets and resource tracking.