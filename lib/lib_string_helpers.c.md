# lib/string_helpers.c - String Formatting and Manipulation Utilities

## Overview

This file provides a comprehensive collection of string formatting, parsing, and manipulation utilities for the Linux kernel. Originally contributed by James Bottomley and enhanced by Intel, it offers essential functions for safe string processing, size formatting, escape sequence handling, and array parsing operations.

## Core Functionality Categories

### 1. Size Formatting
### 2. String Parsing and Arrays  
### 3. Escape Sequence Processing
### 4. String Matching and Comparison
### 5. Memory Operations
### 6. Security and Fortification

## Major Functions

### Size Formatting Functions

#### `string_get_size()` - Human-Readable Size Formatting
```c
int string_get_size(u64 size, u64 blk_size, const enum string_size_units units,
                   char *buf, int len)
```

**Purpose**: Converts byte sizes into human-readable format with appropriate units

**Parameters**:
- `size`: Size to convert (in blocks)
- `blk_size`: Size of each block (use 1 for bytes)
- `units`: Unit system and formatting options
- `buf`: Output buffer (minimum 9 bytes recommended)
- `len`: Buffer length

**Unit Systems**:
- `STRING_UNITS_10`: Powers of 1000 (k, M, G, T, P, E, Z, Y)
- `STRING_UNITS_2`: Powers of 1024 (Ki, Mi, Gi, Ti, Pi, Ei, Zi, Yi)

**Formatting Options**:
- `STRING_UNITS_NO_SPACE`: No space between number and unit
- `STRING_UNITS_NO_BYTES`: No 'B' suffix

**Algorithm - Napier's Method**:
1. **Coefficient Reduction**: Reduces both size and block size to fit in 32 bits
2. **Multiplication**: Performs size * blk_size without overflow
3. **Logarithmic Reduction**: Divides by unit base until manageable
4. **Precision Calculation**: Determines significant figures needed
5. **Rounding**: Applies proper rounding for display

**Examples**:
```c
char buf[16];
string_get_size(1024, 1, STRING_UNITS_2, buf, sizeof(buf));
// Result: "1.00 KiB"

string_get_size(1000, 1, STRING_UNITS_10, buf, sizeof(buf));
// Result: "1.00 kB"

string_get_size(1536, 1, STRING_UNITS_2 | STRING_UNITS_NO_SPACE, buf, sizeof(buf));
// Result: "1.50KiB"
```

### String Parsing Functions

#### `parse_int_array()` - Parse Integer Arrays from Strings
```c
int parse_int_array(const char *buf, size_t count, int **array)
```

**Purpose**: Converts string representation of integers into an allocated array

**Format**: Space or comma-separated integers
**Memory Management**: Caller must free returned array
**Return Format**: Array[0] contains count, Array[1..n] contains values

**Algorithm**:
1. **Count Pass**: Uses `get_options()` to determine number of integers
2. **Allocation**: Allocates array with space for count + values
3. **Parse Pass**: Extracts actual integer values
4. **Result**: Returns allocated array with count as first element

#### `parse_int_array_user()` - User Space Integer Array Parsing
```c
int parse_int_array_user(const char __user *from, size_t count, int **array)
```

**Purpose**: Safe parsing of integer arrays from user space

**Safety Features**:
- **User Memory Validation**: Uses `memdup_user_nul()` 
- **Buffer Management**: Automatic null termination
- **Error Propagation**: Proper error code handling

### Escape Sequence Processing

#### `string_unescape()` - Unescape String Sequences
```c
int string_unescape(char *src, char *dst, size_t size, unsigned int flags)
```

**Purpose**: Converts escaped character sequences back to their original form

**Supported Escape Types**:

**UNESCAPE_SPACE**:
- `\n` → newline
- `\r` → carriage return  
- `\t` → horizontal tab
- `\v` → vertical tab
- `\f` → form feed

**UNESCAPE_OCTAL**:
- `\NNN` → byte with octal value (1-3 digits)
- Example: `\101` → 'A' (ASCII 65)

**UNESCAPE_HEX**:
- `\xHH` → byte with hex value (1-2 digits)
- Example: `\x41` → 'A' (ASCII 65)

**UNESCAPE_SPECIAL**:
- `\"` → double quote
- `\\` → backslash
- `\a` → alert (BEL)
- `\e` → escape character

**Features**:
- **In-Place Operation**: Can operate on same buffer (src == dst)
- **Size Limiting**: Respects destination buffer size
- **Null Termination**: Always null-terminates output

#### `string_escape_mem()` - Escape String Characters
```c
int string_escape_mem(const char *src, size_t isz, char *dst, size_t osz,
                     unsigned int flags, const char *only)
```

**Purpose**: Escapes special characters in strings for safe output

**Escape Modes**:

**ESCAPE_SPACE**: Escape whitespace characters
**ESCAPE_SPECIAL**: Escape special characters  
**ESCAPE_NULL**: Escape null bytes
**ESCAPE_OCTAL**: Use octal notation (\NNN)
**ESCAPE_HEX**: Use hexadecimal notation (\xHH)

**Pass-Through Modes**:
**ESCAPE_NAP**: Don't escape ASCII printable
**ESCAPE_NP**: Don't escape printable characters
**ESCAPE_NA**: Don't escape ASCII characters

**Control Parameters**:
- `only`: Character set to specifically escape/not escape
- `ESCAPE_APPEND`: Modify behavior with `only` parameter

**Algorithm**:
1. **Priority Rules**: Apply escape rules in specific sequence
2. **Pass-Through Check**: Determine if character should pass unchanged
3. **Escape Application**: Apply appropriate escape method
4. **Fallback**: Default passthrough for unmatched characters

#### High-Level Escape Functions

#### `kstrdup_quotable()` - Create Quotable String Copy
```c
char *kstrdup_quotable(const char *src, gfp_t gfp)
```

**Purpose**: Creates escaped copy safe for logging in quotes
**Escapes**: `\f\n\r\t\v\a\e\\\"`
**Method**: Uses hexadecimal escaping
**Memory**: Returns allocated string (caller must free)

#### `kstrdup_quotable_cmdline()` - Safe Command Line Copy
```c
char *kstrdup_quotable_cmdline(struct task_struct *task, gfp_t gfp)
```

**Purpose**: Extracts and escapes process command line for logging

**Process**:
1. **Extraction**: Gets command line from task structure
2. **Null Replacement**: Replaces argument separators with spaces
3. **Escaping**: Makes all characters printable/loggable
4. **Result**: Returns allocated escaped string

#### `kstrdup_quotable_file()` - Safe File Path Copy
```c
char *kstrdup_quotable_file(struct file *file, gfp_t gfp)
```

**Purpose**: Extracts and escapes file path for safe logging

**Error Handling**:
- `<unknown>`: file is NULL
- `<no_memory>`: allocation failed
- `<too_long>`: path exceeds limits

### String Matching Functions

#### `sysfs_streq()` - Sysfs String Comparison
```c
bool sysfs_streq(const char *s1, const char *s2)
```

**Purpose**: String comparison that ignores trailing newlines

**Use Case**: Perfect for sysfs attribute comparison where user input often includes newlines
**Algorithm**: Compares strings while treating trailing `\n` as equivalent to null terminator

#### `match_string()` - Find String in Array
```c
int match_string(const char * const *array, size_t n, const char *string)
```

**Purpose**: Finds exact string match in array of strings

**Return Value**:
- Index of matched string (0-based)
- -EINVAL if no match found

#### `__sysfs_match_string()` - Sysfs-Aware String Matching
```c
int __sysfs_match_string(const char * const *array, size_t n, const char *str)
```

**Purpose**: String matching with sysfs newline handling
**Combines**: Array searching with sysfs string comparison semantics

### Array Management Functions

#### `kfree_strarray()` - Free String Array
```c
void kfree_strarray(char **array, size_t n)
```

**Purpose**: Safely frees array of allocated strings

**Process**:
1. **Individual Strings**: Frees each string in array
2. **Array Structure**: Frees the array pointer container
3. **Null Safety**: Handles NULL pointers gracefully

### Memory Operations

#### `memcpy_and_pad()` - Copy with Padding
```c
void memcpy_and_pad(void *dest, size_t dest_len, const void *src, 
                   size_t count, int pad)
```

**Purpose**: Copies memory and pads remainder with specified byte

**Use Cases**:
- **Buffer Initialization**: Ensure unused areas have known values
- **Protocol Compliance**: Some protocols require specific padding
- **Security**: Clear sensitive data in unused buffer areas

**Algorithm**:
1. **Copy Phase**: Standard memcpy for actual data
2. **Pad Phase**: Fill remaining space with pad byte
3. **Bounds Safety**: Respects destination buffer limits

### Security and Fortification

#### `__fortify_report()` - Buffer Overflow Reporting
```c
void __fortify_report(const u8 reason, const size_t avail, const size_t size)
```

**Purpose**: Reports detected buffer overflow attempts

**Integration**: Works with compiler fortification features
**Reasons**: Different types of detected overflows
**Action**: Logs security violation details

#### `__fortify_panic()` - Critical Security Response
```c
void __fortify_panic(const u8 reason, const size_t avail, const size_t size)
```

**Purpose**: Triggers kernel panic on critical security violations

**Use Case**: When buffer overflow could compromise system integrity
**Response**: Immediate system halt to prevent exploitation

## Helper Functions (Internal)

### Escape Sequence Helpers

#### `unescape_space()` - Process Space Escapes
**Handles**: `\n`, `\r`, `\t`, `\v`, `\f`

#### `unescape_octal()` - Process Octal Escapes
**Format**: `\NNN` where N is octal digit
**Range**: Values 0-255
**Length**: 1-3 digits maximum

#### `unescape_hex()` - Process Hexadecimal Escapes
**Format**: `\xHH` where H is hex digit
**Range**: Values 0-255
**Length**: 1-2 hex digits

#### `unescape_special()` - Process Special Escapes
**Handles**: `\"`, `\\`, `\a`, `\e`

#### Escape Output Helpers

#### `escape_passthrough()` - Pass Character Unchanged
#### `escape_space()` - Escape Whitespace Characters
#### `escape_special()` - Escape Special Characters
#### `escape_null()` - Escape Null Bytes
#### `escape_octal()` - Convert to Octal Escape
#### `escape_hex()` - Convert to Hex Escape

## Performance Considerations

### String Size Formatting
- **Napier's Algorithm**: Prevents 64-bit overflow while maintaining precision
- **Lookup Tables**: Pre-computed unit strings for efficiency
- **Minimal Division**: Reduces expensive division operations

### Memory Management
- **In-Place Operations**: Many functions support in-place transformation
- **Single-Pass Processing**: Most operations complete in one pass
- **Bounded Allocation**: Predictable memory usage patterns

### Escape Processing
- **Early Termination**: Stops on size limits
- **Branch Prediction**: Likely paths optimized
- **Minimal State**: Low memory overhead during processing

## Security Features

### Buffer Overflow Protection
- **Size Validation**: All functions validate buffer limits
- **Null Termination**: Guaranteed null termination
- **Bounds Checking**: Prevents buffer overruns

### Input Validation
- **User Space Safety**: Proper validation of user input
- **Format Validation**: Checks for valid escape sequences
- **Error Propagation**: Clear error indication

### Information Leakage Prevention
- **Controlled Output**: Escaped output prevents injection
- **Safe Logging**: Functions safe for system logs
- **Predictable Behavior**: No undefined behavior paths

## Integration Points

### Sysfs Integration
- **Attribute Handling**: Perfect for sysfs show/store functions
- **User Input Processing**: Handles common user input patterns
- **Newline Handling**: Accounts for sysfs newline conventions

### Logging and Debug
- **Safe Formatting**: Prevents log injection attacks
- **Readable Output**: Human-friendly size formatting
- **Error Reporting**: Clear error message formatting

### Driver Interfaces
- **Parameter Parsing**: Integer array parsing for module parameters
- **Configuration**: String processing for device configuration
- **Status Display**: Size formatting for capacity reporting

## Usage Examples

### Size Formatting
```c
void show_disk_size(u64 bytes) {
    char size_str[16];
    
    string_get_size(bytes, 1, STRING_UNITS_2, size_str, sizeof(size_str));
    printk("Disk size: %s\n", size_str);
}
```

### Safe String Processing
```c
static ssize_t config_store(struct device *dev, const char *buf, size_t count) {
    char *escaped = kstrdup_quotable(buf, GFP_KERNEL);
    if (!escaped)
        return -ENOMEM;
    
    // Process escaped string safely
    dev_info(dev, "Configuration: %s\n", escaped);
    kfree(escaped);
    return count;
}
```

### Integer Array Processing
```c
static int parse_cpu_list(const char *buf) {
    int *cpu_array;
    int ret, i;
    
    ret = parse_int_array(buf, strlen(buf), &cpu_array);
    if (ret)
        return ret;
    
    for (i = 1; i <= cpu_array[0]; i++) {
        printk("CPU: %d\n", cpu_array[i]);
    }
    
    kfree(cpu_array);
    return 0;
}
```

This comprehensive string helper library provides essential functionality for safe, efficient string processing throughout the Linux kernel, with particular attention to security, performance, and ease of use.