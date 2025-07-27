# lib/vsprintf.c - Linux Kernel Formatted String Printing

## Overview

This file implements the complete formatted string printing infrastructure for the Linux kernel, providing printf-family functions (sprintf, snprintf, vsnprintf) and related utilities. Originally based on standard C library implementations, it has evolved into a sophisticated system optimized for kernel use with extensive security features, performance optimizations, and specialized formatters for kernel-specific data types like pointers, IP addresses, and UUIDs.

## Historical Development

### Key Evolution Points
- **Early Linux**: Basic printf implementation adapted from user-space libraries
- **2000s**: Security hardening against format string attacks
- **2010s**: Pointer hashing and information leak prevention
- **2017**: SipHash-based pointer obfuscation (Kernel Address Display Restriction)
- **2020s**: Modern optimizations and specialized format handlers

### Design Philosophy
The kernel vsprintf implementation prioritizes security, performance, and reliability over feature completeness. It provides robust protection against buffer overflows, format string attacks, and information disclosure while maintaining high performance for kernel logging and debugging.

## Core Architecture

### Format Processing Pipeline
```
Format String → Parser → Specifier → Formatter → Output Buffer
      ↓            ↓         ↓          ↓            ↓
["%p %d %s"]  [State Machine] [Spec] [Handler] [Safe Buffer]
```

### Two-Stage Architecture
1. **Format Parsing**: `format_decode()` parses format specifiers into structured data
2. **Output Generation**: Specialized functions convert parsed specifications to formatted output

## Key Data Structures

### Format Specification Structure
```c
struct printf_spec {
    unsigned char flags;        /* Formatting flags (ZEROPAD, LEFT, PLUS, etc.) */
    unsigned char base;         /* Number base (8, 10, or 16) */
    short precision;           /* Precision specification */
    int field_width;          /* Field width */
} __packed;
```

### Format Parser State
```c
struct fmt {
    const char *str;          /* Current position in format string */
    unsigned char state;      /* Parsing state (enum format_state) */
    unsigned char size;       /* Size of numeric arguments */
};

enum format_state {
    FORMAT_STATE_NONE,        /* Regular text */
    FORMAT_STATE_NUM,         /* Numeric format */
    FORMAT_STATE_WIDTH,       /* Width parameter from '*' */
    FORMAT_STATE_PRECISION,   /* Precision parameter from '*' */
    FORMAT_STATE_CHAR,        /* Character format */
    FORMAT_STATE_STR,         /* String format */
    FORMAT_STATE_PTR,         /* Pointer format */
    FORMAT_STATE_PERCENT_CHAR,/* Literal '%' */
    FORMAT_STATE_INVALID,     /* Invalid format */
};
```

### Security and Configuration
```c
/* Pointer hashing security */
static siphash_key_t ptr_key __read_mostly;
static bool filled_random_ptr_key __read_mostly;

/* Security controls */
extern bool no_hash_pointers;     /* Disable pointer hashing */
extern int kptr_restrict;         /* Pointer access restriction levels */
```

## Core Functions

### Primary Entry Points

#### `vsnprintf()` - Main Formatting Function
```c
int vsnprintf(char *buf, size_t size, const char *fmt, va_list args)
```

**Purpose**: Core formatted string creation with buffer size limits

**Security Features**:
- **Buffer overflow protection**: Always checks `str < end` before writing
- **Integer overflow protection**: Validates size parameter against `INT_MAX`
- **Null termination guarantee**: Always null-terminates output even on truncation
- **Wraparound handling**: Detects and handles buffer pointer wraparound

**Algorithm**:
1. **Buffer Setup**: Establish safe buffer boundaries
2. **Format Parsing**: Parse format string character by character
3. **Argument Processing**: Handle variable arguments with type safety
4. **Output Generation**: Generate formatted output with bounds checking
5. **Termination**: Ensure proper null termination

**Key Implementation**:
```c
/* Buffer overflow protection */
if (end < buf) {
    end = ((void *)-1);      /* Handle wraparound */
    size = end - buf;
}

/* Safe character output */
if (str < end)
    *str = c;
++str;  /* Always advance pointer for length calculation */
```

#### `snprintf()` and `sprintf()` - Public Interfaces
```c
int snprintf(char *buf, size_t size, const char *fmt, ...)
int sprintf(char *buf, const char *fmt, ...)
```

**Wrapper Functions**: Provide standard C library compatible interfaces

### Format String Parser

#### `format_decode()` - Format Specifier Parser
```c
static const char *format_decode(const char *fmt, struct printf_spec *spec)
```

**Purpose**: Parse format specifiers into structured data for efficient processing

**Optimization Techniques**:
- **Lookup Tables**: 256-entry array for O(1) format character resolution
- **State Machine**: Efficient parsing of complex format specifications
- **Bit Manipulation**: Uses bit flags for efficient flag combination

**Flag Processing**:
```c
static unsigned char spec_flag(unsigned char c) {
    static const unsigned char spec_flag_array[] = {
        SPEC_CHAR(' ', SPACE),     /* Space flag */
        SPEC_CHAR('#', SPECIAL),   /* Special flag (0x prefix) */
        SPEC_CHAR('+', PLUS),      /* Plus flag */
        SPEC_CHAR('-', LEFT),      /* Left align flag */
        SPEC_CHAR('0', ZEROPAD),   /* Zero padding flag */
    };
    c -= 32;  /* Optimize by subtracting ASCII space */
    return (c < sizeof(spec_flag_array)) ? spec_flag_array[c] : 0;
}
```

**Length Modifier Handling**:
```c
/* Handle double-character modifiers (ll, hh) */
if (p->flags_or_double_size && fmt[0] == fmt[1]) {
    spec->size = p->flags_or_double_size;
    fmt++;  /* Skip second character */
}
```

### Number Formatting Engine

#### `number()` - Core Numeric Formatter
```c
static char *number(char *buf, char *end, unsigned long long num,
                   struct printf_spec spec)
```

**Base-Specific Optimizations**:

**Base 16 (Hexadecimal)**:
```c
if (spec.base == 16) {
    int mask = spec.base - 1;    /* 15 for hex */
    int shift = 4;               /* 4 bits per hex digit */
    do {
        tmp[i++] = hex_asc_upper[num & mask] | locase;
        num >>= shift;
    } while (num);
}
```

**Base 10 (Decimal) - Advanced Optimizations**:
```c
/* Use specialized decimal conversion */
i = put_dec(tmp, num) - tmp;
```

#### Decimal Conversion Optimizations

**Lookup Table for Digit Pairs**:
```c
static const u16 decpair[100] = {
#define _(x) (__force u16) cpu_to_le16(((x % 10) | ((x / 10) << 8)) + 0x3030)
    _( 0), _( 1), _( 2), _( 3), _( 4), _( 5), _( 6), _( 7), _( 8), _( 9),
    _(10), _(11), _(12), _(13), _(14), _(15), _(16), _(17), _(18), _(19),
    /* ... continues to 99 */
};
```

**Division by Multiplication**:
```c
/* Avoid expensive division with mathematical approximation */
q = (r * (u64)0x28f5c29) >> 32;  /* Division by 100 */
q = (r * 0x147b) >> 19;          /* Division by 100 for smaller numbers */
```

### Pointer Security System

#### Pointer Hashing and Obfuscation
```c
static char *ptr_to_id(char *buf, char *end, const void *ptr,
                      struct printf_spec spec)
```

**Security Mechanisms**:
1. **SipHash Cryptographic Hashing**: Uses secure hash function for pointer obfuscation
2. **Random Key Management**: Boot-time random key generation
3. **Capability-Based Access**: Graduated access control for debugging

**Hash Implementation**:
```c
#ifdef CONFIG_64BIT
#define PTR_VAL_BITS 64
static unsigned long ptr_to_hashval(const void *ptr, unsigned long *hashval_out)
{
    unsigned long hashval;
    hashval = (unsigned long)siphash_1u64((u64)ptr, &ptr_key);
    *hashval_out = hashval;
    return hashval & 0xffffffff;  /* Limit to 32-bit output */
}
#else
#define PTR_VAL_BITS 32
static unsigned long ptr_to_hashval(const void *ptr, unsigned long *hashval_out)
{
    unsigned long hashval;
    hashval = (unsigned long)siphash_1u32((u32)ptr, &ptr_key);
    *hashval_out = hashval;
    return hashval;
}
#endif
```

#### Restricted Pointer Access (`%pK`)
```c
static char *restricted_pointer(char *buf, char *end, const void *ptr,
                               struct printf_spec spec)
```

**Access Control Levels**:
- **Level 0**: Hash pointers (default behavior)
- **Level 1**: Show real addresses only to processes with `CAP_SYSLOG` capability
- **Level 2**: Always show zeros (complete restriction)

**Capability Checking**:
```c
const struct cred *cred = current_cred();
if (!has_capability_noaudit(current, CAP_SYSLOG) ||
    !uid_eq(cred->euid, cred->uid) ||
    !gid_eq(cred->egid, cred->gid))
    ptr = NULL;
```

### String Handling System

#### `string()` and `string_nocheck()` - String Formatters
```c
static char *string(char *buf, char *end, const char *s, struct printf_spec spec)
static char *string_nocheck(char *buf, char *end, const char *s,
                           struct printf_spec spec)
```

**Safety Architecture**:
- **Two-tier system**: `string()` validates pointers, `string_nocheck()` assumes safety
- **Null pointer handling**: Returns "(null)" for NULL pointers
- **Precision-based truncation**: Respects precision limits for string length

**Buffer Management**:
```c
/* Safe character copying with bounds checking */
while (len--) {
    if (buf < end)
        *buf = *s;
    ++buf; ++s;
}
```

#### Specialized String Formatters

**Network Address Formatting**:
- **IP Addresses**: IPv4/IPv6 with various format options
- **MAC Addresses**: Multiple separator formats (colon, hyphen, none)
- **Socket Addresses**: Complete sockaddr structure formatting

**System Object Formatting**:
- **UUIDs**: Standard and Microsoft GUID formats
- **Kernel Symbols**: Symbol name lookup with kallsyms integration
- **Resources**: Memory and I/O resource ranges
- **Device Nodes**: Device tree node information

### Advanced Features

#### Escape Sequence Processing
```c
static char *escaped_string(char *buf, char *end, u8 *addr,
                           struct printf_spec spec, const char *fmt)
```

**Escape Modes**:
- `ESCAPE_ANY`: Any characters
- `ESCAPE_SPECIAL`: Special characters  
- `ESCAPE_HEX`: Hexadecimal escaping
- `ESCAPE_NULL`: Null characters
- `ESCAPE_OCTAL`: Octal escaping
- `ESCAPE_NP`: Non-printable characters

#### Format Validation and Security

**Format String Attack Prevention**:
```c
/* %n format disabled for security */
case 'n':
    WARN_ONCE(1, "Please remove unsupported %%%c in format string\n", *fmt);
    return FORMAT_STATE_INVALID;
```

**Bounds Validation**:
```c
#define FIELD_WIDTH_MAX ((1 << 23) - 1)
#define PRECISION_MAX ((1 << 15) - 1)

static void set_field_width(struct printf_spec *spec, int width) {
    spec->field_width = width;
    if (WARN_ONCE(spec->field_width != width, "field width %d too large", width)) {
        spec->field_width = clamp(width, -FIELD_WIDTH_MAX, FIELD_WIDTH_MAX);
    }
}
```

### Performance Optimizations

#### Lookup Table Optimizations
- **Format Specifier Recognition**: 256-entry lookup table for O(1) access
- **Hex Digit Conversion**: Pre-computed ASCII hex tables
- **Decimal Pair Conversion**: Pre-computed 2-digit ASCII pairs

#### Algorithmic Optimizations
- **Bit Manipulation**: Efficient base conversion for powers of 2
- **Mathematical Approximation**: Division replacement with multiplication + shift
- **Cache-Friendly Data Structures**: Aligned and packed structures
- **Minimal Branching**: Lookup tables instead of conditional chains

#### Memory Efficiency
- **Stack-Based Buffers**: Minimal heap usage
- **Packed Structures**: `printf_spec` optimized to 8 bytes
- **Zero-Copy Operations**: Direct buffer manipulation

## Security Features

### Buffer Overflow Prevention
- **Comprehensive Boundary Checking**: Every write operation validated
- **End Pointer Arithmetic**: Safe buffer end calculations
- **Wraparound Detection**: Integer overflow protection

### Information Disclosure Prevention
- **Default Pointer Hashing**: Cryptographic obfuscation of kernel addresses
- **Capability-Based Access**: Graduated access control for sensitive information
- **Error Pointer Handling**: Safe handling of kernel error values

### Format String Attack Mitigation
- **Disabled %n**: Prevents format string write attacks
- **Controlled Parsing**: Robust parser with validation
- **Warning System**: Runtime detection of suspicious usage

## Integration Points

### Kernel Logging Integration
- **printk**: Primary interface for kernel message logging
- **Device Drivers**: Driver debugging and error reporting
- **Subsystem Logging**: Filesystem, network, memory management logging

### Debug and Tracing Integration
- **ftrace**: Function tracing with formatted output
- **Dynamic Debug**: Runtime debug message control
- **Kernel Debugging**: GDB and other debugger support

### Security Framework Integration
- **Capability System**: Integration with Linux capability model
- **LSM Hooks**: Linux Security Module integration potential
- **Audit System**: Security audit message formatting

This comprehensive vsprintf implementation demonstrates sophisticated systems programming with extensive security hardening, performance optimization, and robust error handling suitable for critical kernel-level formatted output operations.