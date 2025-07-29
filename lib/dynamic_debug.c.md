# dynamic_debug.c - Runtime Configurable Debug Infrastructure

## Overview

The `dynamic_debug.c` file implements Linux's dynamic debug infrastructure, which allows `pr_debug()` and `dev_dbg()` calls to be enabled or disabled at runtime. This provides a powerful debugging mechanism that can be selectively activated without recompiling the kernel, making it invaluable for system debugging and development.

## File Location
- **Path**: `lib/dynamic_debug.c`
- **Purpose**: Runtime configurable debug output system
- **Authors**: 
  - Jason Baron (jbaron@redhat.com) - 2008
  - Greg Banks (gnb@melbourne.sgi.com)
  - Various contributors for enhancements

## Key Features

### Runtime Configuration
- **Selective Activation**: Enable/disable debug statements based on module, file, function, or line
- **No Recompilation**: Changes take effect immediately without kernel rebuild
- **Fine-Grained Control**: Individual debug statements can be controlled
- **Pattern Matching**: Wildcard support for bulk operations

### Output Formatting
- **Flexible Output**: Configurable inclusion of module name, function, source file, line number, thread ID
- **Custom Formats**: Support for different output format combinations
- **Class-Based Grouping**: Logical grouping of related debug statements

## Core Data Structures

### ddebug_table
```c
struct ddebug_table {
    struct list_head link, maps;    // Linked list nodes
    const char *mod_name;           // Module name
    unsigned int num_ddebugs;       // Number of debug sites
    struct _ddebug *ddebugs;        // Array of debug descriptors
};
```

Represents a collection of debug sites for a single module.

### ddebug_query
```c
struct ddebug_query {
    const char *filename;           // Source filename filter
    const char *module;             // Module name filter
    const char *function;           // Function name filter
    const char *format;             // Format string filter
    const char *class_string;       // Debug class filter
    unsigned int first_lineno, last_lineno; // Line number range
};
```

Specifies criteria for selecting debug sites to modify.

### ddebug_iter
```c
struct ddebug_iter {
    struct ddebug_table *table;     // Current table
    int idx;                        // Current index within table
};
```

Iterator for traversing debug sites across all modules.

### flag_settings
```c
struct flag_settings {
    unsigned int flags;             // Flags to set
    unsigned int mask;              // Which flags to modify
};
```

Specifies which debug flags to modify and their new values.

## Global State Management

### Protection and Lists
```c
static DEFINE_MUTEX(ddebug_lock);          // Global synchronization
static LIST_HEAD(ddebug_tables);           // List of all debug tables
```

### Linker Sections
```c
extern struct _ddebug __start___dyndbg[];   // Start of debug section
extern struct _ddebug __stop___dyndbg[];    // End of debug section
extern struct ddebug_class_map __start___dyndbg_classes[]; // Class maps start
extern struct ddebug_class_map __stop___dyndbg_classes[];  // Class maps end
```

## Module Parameters

### Verbosity Control
```c
static int verbose;
module_param(verbose, int, 0644);
```

**Verbosity Levels**:
- **0**: Off (default) - No processing messages
- **1**: Module add/remove operations
- **2**: Control summary operations  
- **3**: Query parsing details
- **4**: Per-site change notifications

## Flag Management

### Debug Output Flags
```c
static const struct { unsigned flag:8; char opt_char; } opt_array[] = {
    { _DPRINTK_FLAGS_PRINT, 'p' },           // Enable printing
    { _DPRINTK_FLAGS_INCL_MODNAME, 'm' },    // Include module name
    { _DPRINTK_FLAGS_INCL_FUNCNAME, 'f' },   // Include function name
    { _DPRINTK_FLAGS_INCL_SOURCENAME, 's' }, // Include source filename
    { _DPRINTK_FLAGS_INCL_LINENO, 'l' },     // Include line number
    { _DPRINTK_FLAGS_INCL_TID, 't' },        // Include thread ID
    { _DPRINTK_FLAGS_NONE, '_' },            // No flags (placeholder)
};
```

### Flag Description
```c
struct flagsbuf { char buf[ARRAY_SIZE(opt_array)+1]; };
```

Buffer for representing flag states as human-readable strings.

## Core Functions

### Flag Management

#### `ddebug_describe_flags(unsigned int flags, struct flagsbuf *fb)`
Converts debug flags to human-readable string representation.
- **Parameters**: Flag bits and output buffer
- **Returns**: String representation of active flags
- **Format**: Single character per flag ('p', 'm', 'f', 's', 'l', 't', or '_' for none)

### Path Processing

#### `trim_prefix(const char *path)`
Removes source tree prefix from file paths for cleaner output.
- **Parameters**: Full source file path
- **Returns**: Path relative to source root
- **Purpose**: Makes debug output more readable by removing build-specific prefixes

### Verbosity Control

#### Verbose Printing Macros
```c
#define vnpr_info(lvl, fmt, ...)     // Print if verbose >= lvl
#define vpr_info(fmt, ...)          // Print if verbose >= 1
#define v2pr_info(fmt, ...)         // Print if verbose >= 2
#define v3pr_info(fmt, ...)         // Print if verbose >= 3
#define v4pr_info(fmt, ...)         // Print if verbose >= 4
```

#### `vpr_info_dq(const struct ddebug_query *query, const char *msg)`
Prints detailed query information for debugging the debug system itself.
- **Parameters**: Query structure and message
- **Output**: Formatted query details including function, file, module, format, line range, and class
- **Verbosity**: Requires verbose level 3 or higher

### Class Management

#### `ddebug_find_valid_class(struct ddebug_table const *dt, const char *class_string, int *class_id)`
Finds and validates debug class within a module's class maps.
- **Parameters**: Debug table, class name, output class ID
- **Returns**: Class map pointer or NULL if not found
- **Logic**:
  1. Iterates through module's class maps
  2. Uses `match_string()` to find class name
  3. Calculates absolute class ID from base + index
  4. Sets class_id to -ENOENT if not found

### Query Processing

#### `ddebug_change(const struct ddebug_query *query, struct flag_settings *modifiers)`
Core function that applies flag changes to matching debug sites.
- **Parameters**: Query criteria and flag modifications
- **Returns**: Number of sites modified
- **Protection**: Takes `ddebug_lock` mutex
- **Process**:
  1. **Module Filtering**: Checks module name against query
  2. **Class Validation**: Validates debug class if specified
  3. **Site Iteration**: Examines each debug site in matching modules
  4. **Criteria Matching**: Applies filename, function, format, and line filters
  5. **Flag Application**: Updates flags according to modifiers
  6. **Logging**: Reports changes if verbose mode enabled

## Filtering and Matching

### Module Matching
- **Wildcard Support**: Uses `match_wildcard()` for flexible module name matching
- **Exact Match**: Direct string comparison when no wildcards present

### Class Constraints
- **Class-Specific**: Only affects sites with matching debug class
- **Class Exclusion**: When no class specified, avoids modifying class-specific sites
- **Validation**: Ensures class exists in module before applying changes

### Multi-Criteria Filtering
Debug sites must match ALL specified criteria:
- **Filename**: Source file path matching
- **Function**: Function name matching  
- **Format**: Debug format string matching
- **Line Range**: Line number within specified range
- **Class**: Debug class membership

## Memory Management

### Dynamic Allocation
- **Table Structures**: Debug tables allocated dynamically during module loading
- **Reference Counting**: Proper cleanup when modules are unloaded
- **Iterator Safety**: Safe traversal even during concurrent modifications

### Linker Integration
- **Section-Based**: Debug descriptors stored in special linker sections
- **Automatic Discovery**: Kernel automatically finds debug sites via linker symbols
- **Zero Runtime Cost**: Disabled debug sites have minimal performance impact

## Performance Characteristics

### Runtime Overhead
- **Disabled Sites**: Near-zero overhead when debug disabled
- **Jump Labels**: Uses jump label optimization for minimal impact
- **Batch Operations**: Efficient bulk enable/disable operations

### Memory Efficiency
- **Compact Storage**: Debug descriptors stored efficiently
- **Lazy Loading**: Tables loaded only when modules contain debug sites
- **Cache Friendly**: Data structures optimized for cache locality

## Integration Points

### Module System
- **Automatic Registration**: Debug sites registered during module load
- **Cleanup**: Proper cleanup during module unload
- **Namespace**: Per-module debug namespaces

### Control Interfaces
- **debugfs**: Primary user interface via `/sys/kernel/debug/dynamic_debug/control`
- **Module Parameters**: Boot-time and runtime configuration
- **sysctl**: System-wide debug configuration

### Output System
- **printk Integration**: Uses standard kernel logging infrastructure
- **Rate Limiting**: Integrates with printk rate limiting
- **Log Levels**: Respects kernel log level settings

## Security Considerations

### Access Control
- **Root Only**: Debug control typically requires root privileges
- **debugfs Security**: Relies on debugfs mount permissions
- **Information Leakage**: Debug output may contain sensitive information

### Resource Protection
- **Mutex Protection**: Global lock prevents concurrent modifications
- **Safe Iteration**: RCU or similar protection for list traversal
- **Memory Bounds**: Proper bounds checking for all operations

## Debugging the Debug System

### Self-Debugging
- **Verbose Modes**: Multiple verbosity levels for troubleshooting
- **Query Tracing**: Detailed logging of query processing
- **Flag Tracing**: Reports flag changes as they occur

### Common Issues
- **Module Loading**: Debug sites not appearing (module not loaded)
- **Pattern Matching**: Wildcards not working as expected
- **Class Confusion**: Class-based filtering excluding expected sites
- **Permission Issues**: Access denied to control interface

This dynamic debug infrastructure provides a powerful and flexible debugging mechanism that is essential for kernel development and system troubleshooting, allowing developers to get detailed insights into kernel behavior without the overhead of always-on debug output.