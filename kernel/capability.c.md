# capability.c - Linux Capability System Implementation

## Overview

The `capability.c` file implements the Linux capability system, which provides fine-grained privilege control by dividing the traditional superuser privileges into distinct, manageable capabilities. This system allows processes to hold specific privileges without requiring full root access.

## File Location
- **Path**: `kernel/capability.c`
- **License**: GPL-2.0
- **Authors**: 
  - Andrew Main (zefram@fysh.org) - 1997
  - Andrew G. Morgan (morgan@kernel.org) - Integration
  - Robert M. Love (rml@tech9.net) - 2002 Cleanup

## Core Concepts

### Capability System Purpose
The capability system replaces the traditional binary root/non-root privilege model with a set of individual capabilities that can be independently granted or revoked. This enables:
- **Principle of Least Privilege**: Processes get only needed capabilities
- **Security Enhancement**: Reduces attack surface by limiting privileges
- **Granular Control**: Fine-grained permission management

### Global Configuration

#### File Capabilities Control
```c
int file_caps_enabled = 1;
```
Global flag controlling whether file-based capabilities are enabled.

#### Boot Parameter
- **Parameter**: `no_file_caps`
- **Function**: `file_caps_disable()`
- **Effect**: Disables file capability support system-wide

## Capability Versions

### Version Evolution
The capability system has evolved through three versions:

#### Version 1 (_LINUX_CAPABILITY_VERSION_1)
- **Legacy Support**: 32-bit capabilities
- **Limitations**: Limited capability space
- **Status**: Deprecated but supported for compatibility
- **Warning**: Generates legacy capability use warning

#### Version 2 (_LINUX_CAPABILITY_VERSION_2)  
- **Enhanced Space**: Extended capability space
- **Issues**: Insecure usage patterns possible
- **Status**: Deprecated due to security concerns
- **Warning**: Generates deprecated v2 warning

#### Version 3 (_LINUX_CAPABILITY_VERSION_3)
- **Current Standard**: Modern capability interface
- **Security**: Addresses v2 security issues
- **Compatibility**: Functionally equivalent to v2 with header protection

### Version Validation

#### `cap_validate_magic(cap_user_header_t header, unsigned *tocopy)`
Validates capability version and determines data structure size.
- **Parameters**: User header and copy size output
- **Returns**: 0 on success, negative error on failure
- **Process**:
  1. Reads version from user header
  2. Validates against known versions
  3. Issues appropriate warnings for deprecated versions
  4. Sets copy size for data transfer
  5. Returns kernel version on unknown version

#### Warning Functions

##### `warn_legacy_capability_use()`
Issues one-time warning for Version 1 capability usage.
- **Message**: Identifies process using 32-bit capabilities
- **Frequency**: Once per boot to avoid log spam

##### `warn_deprecated_v2()`
Issues one-time warning for Version 2 capability usage.
- **Security Focus**: Highlights potential insecure usage
- **Recommendation**: Upgrade to libcap 2.10+ or recompile

## Core System Calls

### capget() - Get Process Capabilities

#### `SYSCALL_DEFINE2(capget, cap_user_header_t, header, cap_user_data_t, dataptr)`
Retrieves capability sets for a specified process.

**Parameters**:
- `header`: Contains version and target PID
- `dataptr`: Output buffer for capability data

**Process Flow**:
1. **Version Validation**: Validates capability version
2. **PID Extraction**: Gets target process ID from header
3. **Target Resolution**: Finds target process or uses current
4. **Capability Retrieval**: Gets effective, inheritable, and permitted sets
5. **Format Conversion**: Converts to user-space format
6. **Data Copy**: Copies to user space with appropriate size

**Return Values**:
- `0`: Success
- `-EFAULT`: Memory access error
- `-EINVAL`: Invalid parameters
- `-ESRCH`: Process not found

#### `cap_get_target_pid(pid_t pid, kernel_cap_t *pEp, kernel_cap_t *pIp, kernel_cap_t *pPp)`
Internal function to retrieve capabilities for a specific process.

**Parameters**:
- `pid`: Target process ID (0 for current)
- `pEp`: Effective capabilities output
- `pIp`: Inheritable capabilities output  
- `pPp`: Permitted capabilities output

**Logic**:
- **Current Process**: Direct capability access (no locking needed)
- **Other Process**: RCU-protected lookup and access
- **Security Hook**: Uses `security_capget()` for LSM integration

### capset() - Set Process Capabilities

#### `SYSCALL_DEFINE2(capset, cap_user_header_t, header, const cap_user_data_t, data)`
Sets capability sets for the current process.

**Security Model**:
- **Self-Only**: Only current process can modify its own capabilities
- **No Concurrency Issues**: Single process modification eliminates races
- **Privilege Requirements**: Must have appropriate capabilities to set others

## Data Structure Handling

### Capability Format Conversion

#### `mk_kernel_cap(u32 low, u32 high)`
Creates kernel capability structure from user-space 32-bit values.
- **Parameters**: Low and high 32-bit capability values
- **Returns**: Combined 64-bit kernel capability
- **Validation**: Applies `CAP_VALID_MASK` to ensure valid bits

#### Legacy Format Handling
```c
kdata[0].effective   = pE.val; 
kdata[1].effective   = pE.val >> 32;
kdata[0].permitted   = pP.val; 
kdata[1].permitted   = pP.val >> 32;
kdata[0].inheritable = pI.val; 
kdata[1].inheritable = pI.val >> 32;
```

**64-bit to 32-bit Conversion**:
- Splits 64-bit kernel capabilities into two 32-bit user values
- Maintains compatibility with older user-space libraries
- Handles endianness correctly across architectures

### Backward Compatibility

#### Capability Dropping Behavior
When older applications use newer kernel:
- **Silent Dropping**: Upper capability bits silently dropped
- **Fail-Safe Design**: Older applications continue working
- **Upgrade Path**: Newer libcap versions access full capability space
- **Alternative Considered**: Returning `-ERANGE` would break legacy apps

## Security Integration

### LSM Framework Integration
- **Hook Points**: `security_capget()` and `security_capset()`
- **Policy Enforcement**: LSMs can override capability decisions
- **Audit Integration**: Capability operations can be audited

### Process Context Safety
- **Single Process Model**: Only current process modifies its capabilities
- **Lock-Free Current**: No locking needed for current process access
- **RCU for Others**: Safe access to other processes via RCU

## Memory Safety

### User Space Access
- **Validation**: All user pointers validated before access
- **Copy Operations**: Safe copying with `copy_to_user()`/`copy_from_user()`
- **Error Handling**: Proper error propagation for memory faults

### RCU Protection
- **Process Lookup**: `find_task_by_vpid()` under RCU read lock
- **Safe Dereferencing**: RCU ensures process structure validity
- **Proper Cleanup**: RCU read unlock in all code paths

## Configuration and Debugging

### Compile-Time Configuration
- **CONFIG_MULTIUSER**: Enables multi-user capability support
- **Conditional Compilation**: Core functions only built when needed

### Runtime Configuration
- **Boot Parameters**: `no_file_caps` to disable file capabilities
- **Sysctl Interface**: Runtime capability system tuning
- **Namespace Integration**: Per-namespace capability management

### Debug Support
- **Warning Messages**: Clear messages for deprecated usage
- **Process Identification**: Includes process name in warnings
- **One-Time Warnings**: Prevents log spam with `pr_info_once()`

## Error Handling

### Comprehensive Error Cases
- **Invalid Magic**: Unknown capability version numbers
- **Memory Faults**: User space memory access failures  
- **Process Not Found**: Target process lookup failures
- **Permission Denied**: Insufficient privileges for operations

### Fail-Safe Design
- **Capability Dropping**: Graceful handling of version mismatches
- **Legacy Support**: Maintains compatibility with older applications
- **Security Default**: Fails securely when operations cannot complete

## Performance Considerations

### Lock-Free Current Process
- **No Synchronization**: Current process capability access requires no locks
- **Scalability**: Multiple threads can access capabilities concurrently
- **Efficiency**: Direct access without kernel synchronization overhead

### RCU for Cross-Process
- **Read-Mostly**: Process lookups are typically read-only operations
- **Scalability**: Multiple concurrent capability queries
- **Low Overhead**: RCU provides efficient protection

### Memory Efficiency
- **Stack Allocation**: Temporary structures allocated on stack
- **Minimal Copying**: Only necessary data copied between spaces
- **Version-Specific**: Copy size adjusted based on capability version

This implementation provides a robust, backward-compatible capability system that balances security, performance, and compatibility requirements while providing a foundation for fine-grained privilege management in modern Linux systems.