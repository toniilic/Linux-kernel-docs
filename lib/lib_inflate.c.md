# lib/inflate.c - DEFLATE Decompression Implementation

## Overview

This file implements the DEFLATE decompression algorithm (RFC 1951) used for decompressing gzip-compressed data in the Linux kernel. Originally based on Mark Adler's inflate.c and adapted for kernel use, it provides a crucial component for the kernel's ability to handle compressed data during boot and runtime.

## Historical Context

- **Original Author**: Mark Adler (1992-1993, not copyrighted)
- **Kernel Adaptation**: Hannu Savolainen (1993) for Linux boot
- **Embedded Systems Support**: Nicolas Pitre (1999) for ROM/Flash execution
- **Base Version**: gzip-1.0.3 compatible

## DEFLATE Algorithm Overview

DEFLATE combines LZ77 compression with Huffman coding:

### LZ77 Component
- **Sliding Window**: 32KB backward reference window
- **Match Search**: Finds repeated byte sequences up to 258 bytes long
- **Distance Encoding**: References previous data up to 32KB back
- **Minimum Match**: 3 bytes (shorter sequences encoded as literals)

### Huffman Coding Component
- **Dual Huffman Trees**: Separate trees for literals/lengths and distances
- **Adaptive Coding**: Trees can be customized per block or use fixed tables
- **Efficient Decoding**: Multi-level table lookup for speed

## Block Types

### Type 0: Stored (Uncompressed)
- **Usage**: When data is incompressible
- **Format**: Raw bytes with length header
- **Structure**: [Length][~Length][Data...]
- **Alignment**: Byte-aligned data

### Type 1: Fixed Huffman
- **Usage**: Small blocks where custom trees aren't worth the overhead
- **Literal Codes**: 
  - 0-143: 8 bits
  - 144-255: 9 bits
  - 256-279: 7 bits
  - 280-287: 8 bits
- **Distance Codes**: All 5 bits

### Type 2: Dynamic Huffman
- **Usage**: Most compressed blocks for optimal compression
- **Header**: Custom Huffman table definitions
- **Encoding**: Tables themselves are Huffman-encoded
- **Optimization**: Tailored to actual data distribution

## Core Data Structures

### `struct huft` - Huffman Table Entry
```c
struct huft {
    uch e;                /* number of extra bits or operation */
    uch b;                /* number of bits in this code or subcode */
    union {
        ush n;            /* literal, length base, or distance base */
        struct huft *t;   /* pointer to next level of table */
    } v;
};
```

**Field Meanings**:
- `e == 15`: End of block (EOB)
- `e == 16`: Literal value
- `16 < e < 32`: Pointer to next table level (codes `e-16` bits)
- `e == 99`: Invalid/unused code
- `0 <= e <= 13`: Extra bits needed for length/distance

### Global State Variables
- `bb`: Bit buffer for input stream
- `bk`: Number of valid bits in bit buffer
- `wp`: Current window position
- `slide`: 32KB sliding window buffer

## Key Functions

### `huft_build()` - Huffman Table Construction
```c
STATIC int INIT huft_build(unsigned *b, unsigned n, unsigned s,
                          const ush *d, const ush *e,
                          struct huft **t, int *m)
```

**Purpose**: Builds multi-level Huffman decoding tables from code lengths

**Parameters**:
- `b[]`: Array of code lengths for each symbol
- `n`: Number of codes
- `s`: Number of simple-valued codes (0..s-1)
- `d[]`: Base values for non-simple codes
- `e[]`: Extra bits for non-simple codes
- `t`: Returns pointer to constructed table
- `m`: Maximum/actual lookup bits

**Algorithm**:
1. **Count Analysis**: Count codes of each length
2. **Validation**: Check for valid/complete code sets
3. **Table Sizing**: Determine optimal table sizes
4. **Construction**: Build multi-level lookup tables
5. **Linking**: Connect table levels with pointers

**Return Values**:
- `0`: Success
- `1`: Incomplete table (still usable)
- `2`: Invalid input (over-subscribed)
- `3`: Out of memory

### `inflate_codes()` - Core Decompression
```c
STATIC int INIT inflate_codes(struct huft *tl, struct huft *td, int bl, int bd)
```

**Purpose**: Decompresses data using provided Huffman tables

**Process**:
1. **Literal Decoding**: Look up in literal/length table
2. **Length Extraction**: Get match length with extra bits
3. **Distance Decoding**: Look up in distance table  
4. **Distance Calculation**: Compute backward reference
5. **Copy Operation**: Copy matched bytes to output
6. **Window Management**: Handle circular buffer wrapping

**Optimizations**:
- **Fast Path**: Direct memcpy() for non-overlapping copies
- **Slow Path**: Byte-by-byte for overlapping regions
- **Bit Caching**: Minimize bit extraction operations

### `inflate_fixed()` - Fixed Huffman Decompression
**Setup Process**:
1. **Literal Table**: Build 288-entry table with fixed lengths
2. **Distance Table**: Build 30-entry table with 5-bit codes
3. **Decompression**: Call `inflate_codes()` with fixed tables
4. **Cleanup**: Free temporary structures

### `inflate_dynamic()` - Dynamic Huffman Decompression
**Header Parsing**:
1. **Count Fields**: Read literal, distance, and bit-length code counts
2. **Bit-Length Codes**: Read code lengths for the meta-alphabet
3. **Meta-Table**: Build Huffman table for decoding the main tables
4. **Main Tables**: Decode actual literal/length and distance code lengths
5. **Table Construction**: Build final Huffman tables
6. **Decompression**: Process compressed data

**Complexity Handling**:
- **Run-Length Encoding**: Codes 16, 17, 18 for repeated lengths
- **Variable Bit Fields**: Different bit counts for different ranges
- **Error Recovery**: Graceful handling of incomplete trees

### `inflate_stored()` - Uncompressed Block Handling
**Process**:
1. **Alignment**: Skip to byte boundary
2. **Length Reading**: Read length and its complement
3. **Validation**: Verify length == ~complement
4. **Data Copy**: Direct byte-by-byte copy to output
5. **Window Update**: Maintain sliding window state

## Memory Management

### Custom malloc Implementation
```c
static void *malloc(int size)
static void free(void *where)
```

**Features**:
- **Simple Allocator**: Linear allocation from fixed pool
- **Alignment**: 4-byte alignment for performance
- **Bounds Checking**: Prevents overflow past memory limits
- **Reference Counting**: Tracks allocations for cleanup
- **Fallback**: Uses kmalloc/kfree when `NO_INFLATE_MALLOC` defined

**Memory Pool**:
- `free_mem_ptr`: Start of available memory
- `free_mem_end_ptr`: End of memory pool
- `malloc_ptr`: Current allocation pointer
- `malloc_count`: Number of active allocations

## Bit Stream Handling

### Bit Manipulation Macros
```c
#define NEEDBITS(n) {while(k<(n)){b|=((ulg)NEXTBYTE())<<k;k+=8;}}
#define DUMPBITS(n) {b>>=(n);k-=(n);}
#define NEXTBYTE()  ({ int v = get_byte(); if (v < 0) goto underrun; (uch)v; })
```

**NEEDBITS(n)**:
- Ensures at least `n` bits available in bit buffer
- Fetches bytes from input stream as needed
- Handles input underrun conditions

**DUMPBITS(n)**:
- Removes `n` bits from the bit buffer
- Updates bit count accordingly

**NEXTBYTE()**:
- Reads next byte from input stream
- Handles end-of-input gracefully
- Jumps to underrun handler on error

### Bit Buffer Management
- **Buffer Size**: 32 bits (ulg type)
- **Little-Endian**: Bits added to high end, consumed from low end
- **Byte Alignment**: Special handling for stored blocks
- **Lookahead**: Efficient bit consumption with minimal I/O

## Gzip Format Support

### `gunzip()` - Complete Gzip Decompression
**Header Processing**:
1. **Magic Numbers**: Verify gzip signature (0x1f, 0x8b)
2. **Compression Method**: Ensure DEFLATE (method 8)
3. **Flags Processing**: Handle various gzip options
4. **Metadata Skipping**: Skip timestamps, extra fields, names, comments
5. **Security Checks**: Reject encrypted or invalid files

**Trailer Validation**:
1. **CRC-32 Check**: Verify data integrity
2. **Length Check**: Confirm uncompressed size
3. **Error Reporting**: Detailed error messages

**Supported Flags**:
- `EXTRA_FIELD`: Extra data field
- `ORIG_NAME`: Original filename
- `COMMENT`: File comment
- **Rejected**: `ENCRYPTED`, `CONTINUATION`, `RESERVED`

## CRC-32 Implementation

### `makecrc()` - Table Generation
**Algorithm**: Standard CRC-32 (IEEE 802.3 polynomial)
**Polynomial**: `0xEDB88320` (reversed 0x04C11DB7)
**Implementation**: 
- **Table-Driven**: 256-entry lookup table
- **Initialization**: Computed at runtime for ROM compatibility
- **Standard**: Compatible with gzip/zip CRC-32

**Usage**:
- **Incremental**: CRC updated during decompression
- **Validation**: Final CRC compared with gzip trailer
- **Error Detection**: Catches data corruption

## Performance Optimizations

### Table Lookup Strategy
- **Multi-Level Tables**: Balance memory vs. speed
- **Optimal Sizing**: `lbits=9` for literals, `dbits=6` for distances
- **Fast Paths**: Direct table lookup for common codes
- **Fallback**: Chain traversal for longer codes

### Memory Access Patterns
- **Sequential Reads**: Minimize random access
- **Bulk Copies**: Use `memcpy()` when possible
- **Cache Friendly**: Localized data access
- **Window Reuse**: Exploit temporal locality

### Compiler Optimizations
- **noinline**: Prevent excessive stack usage in gcc-3.5
- **Register Variables**: Guide compiler optimization
- **Loop Unrolling**: Manual optimization where beneficial

## Error Handling

### Error Codes
- **1**: Invalid compressed format (Huffman errors)
- **2**: Invalid compressed format (structure errors)  
- **3**: Out of memory
- **4**: Input underrun/truncation

### Validation Checks
- **Code Completeness**: Huffman tables must be complete
- **Length Validation**: Prevent buffer overruns
- **Range Checks**: Ensure valid table indices
- **CRC Verification**: Detect data corruption

### Recovery Strategies
- **Graceful Degradation**: Continue with incomplete tables when safe
- **Resource Cleanup**: Free allocated memory on errors
- **Error Propagation**: Clear error reporting to callers

## Integration Points

### Kernel Boot Support
- **Early Decompression**: Works before full kernel initialization
- **Memory Constraints**: Operates within limited memory pools
- **No Standard Library**: Self-contained implementation
- **ROM Compatibility**: Supports execution from read-only memory

### Architecture Support
- **Watchdog Integration**: `ARCH_HAS_DECOMP_WDOG` for long operations
- **Endianness**: Handles different byte orders correctly
- **Alignment**: Respects architecture alignment requirements

### Interface Dependencies
**Required External Functions**:
- `get_byte()`: Input stream interface
- `flush_window()`: Output buffer management
- `error()`: Error reporting
- `memzero()`, `memcpy()`: Basic memory operations

**Global Variables**:
- `outcnt`: Output byte count
- `bytes_out`: Total bytes decompressed
- `inptr`: Input stream pointer

## Security Considerations

### Buffer Overflow Protection
- **Bounds Checking**: All array accesses validated
- **Size Limits**: Maximum table sizes enforced
- **Length Validation**: Input lengths checked against buffer sizes
- **Window Management**: Circular buffer prevents overruns

### Input Validation
- **Magic Number Verification**: Prevents processing invalid data
- **Flag Validation**: Rejects unsupported or dangerous options
- **Size Consistency**: Ensures header/trailer consistency
- **Format Compliance**: Strict adherence to DEFLATE specification

### Memory Safety
- **Allocation Tracking**: Prevents memory leaks
- **Initialization**: All structures properly initialized
- **Cleanup Paths**: Resources freed on all exit paths
- **Stack Protection**: Limited recursion and stack usage

This implementation provides a robust, efficient, and secure foundation for DEFLATE decompression in the Linux kernel, supporting both boot-time and runtime decompression needs while maintaining compatibility with standard gzip format.