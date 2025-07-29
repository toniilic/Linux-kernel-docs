# commoncap.c - Common Capability Implementation for LSM

## Overview

The `commoncap.c` file implements the common capability functionality required by the Linux Security Module (LSM) framework. This file provides the default capability-based security checks that can be used by security modules or as standalone security enforcement. It handles capability checking, privilege escalation during exec, and various security-sensitive operations.

## File Location
- **Path**: `security/commoncap.c`
- **License**: GPL-2.0-or-later
- **Purpose**: Common capability implementation for LSM hooks

## Core Concepts

### Capability-Based Access Control
The file implements POSIX-style capabilities that divide traditional superuser privileges into discrete, manageable units:
- **Effective Capabilities**: Currently active capabilities
- **Permitted Capabilities**: Capabilities that can be made effective
- **Inheritable Capabilities**: Capabilities passed to child processes
- **Bounding Set**: Maximum capabilities a process can acquire

### User Namespace Integration
Full integration with user namespaces provides:
- **Namespace-Aware Checks**: Capabilities checked within appropriate namespace context
- **Privilege Delegation**: Namespace owners have capabilities over their namespace
- **Hierarchical Model**: Parent namespaces have authority over children

## Warning and Compatibility

### Mixed Setuid/Capability Warning
```c
static void warn_setuid_and_fcaps_mixed(const char *fname)
{
    static int warned;
    if (!warned) {
        printk(KERN_INFO "warning: `%s' has both setuid-root and"
            " effective capabilities. Therefore not raising all"
            " capabilities.\n", fname);
        warned = 1;
    }
}
```

**Purpose**: Warns about potentially confusing security configurations
**Trigger**: Binary has both setuid-root and file capabilities
**Behavior**: Only file capabilities applied, not full root privileges
**Frequency**: One warning per boot to avoid log spam

## Core Capability Functions

### Capability Checking

#### `cap_capable_helper(const struct cred *cred, struct user_namespace *target_ns, const struct user_namespace *cred_ns, int cap)`
Internal helper for checking capabilities across namespaces.

**Parameters**:
- `cred`: Credentials to check
- `target_ns`: Namespace of resource being accessed
- `cred_ns`: Namespace of the credentials
- `cap`: Capability to check

**Returns**: 0 if capability present, -EPERM if not

**Algorithm**:
1. **Namespace Traversal**: Walks up namespace hierarchy from target to credential namespace
2. **Direct Match**: If namespaces match, checks effective capabilities
3. **Level Check**: Ensures we're not checking at inappropriate level
4. **Owner Check**: Namespace owners have all capabilities over their namespace
5. **Parent Authority**: Capabilities in parent namespace apply to children

**Key Logic**:
```c
for (;;) {
    if (likely(ns == cred_ns))
        return cap_raised(cred->cap_effective, cap) ? 0 : -EPERM;
    
    if (ns->level <= cred_ns->level)
        return -EPERM;
    
    if ((ns->parent == cred_ns) && uid_eq(ns->owner, cred->euid))
        return 0;
    
    ns = ns->parent;
}
```

#### `cap_capable(const struct cred *cred, struct user_namespace *target_ns, int cap, unsigned int opts)`
Main capability checking function with tracing support.

**Parameters**:
- `cred`: Credentials to check
- `target_ns`: Target namespace
- `cap`: Capability to check
- `opts`: Options (currently unused)

**Returns**: 0 if permitted, negative error if denied

**Important Note**: Has **reverse semantics** compared to kernel's `capable()` functions:
- `cap_capable()` returns 0 for success (has capability)
- `capable()` returns true (1) for success

**Tracing**: Includes capability check tracing via `trace_cap_capable()`

### System Operations

#### `cap_settime(const struct timespec64 *ts, const struct timezone *tz)`
Controls permission to set system time and timezone.

**Parameters**:
- `ts`: Time to set
- `tz`: Timezone to set

**Returns**: 0 if permitted, -EPERM if denied

**Requirement**: Requires `CAP_SYS_TIME` capability

**Usage**: Called by settimeofday() and related system calls

### Process Tracing (ptrace)

#### `cap_ptrace_access_check(struct task_struct *child, unsigned int mode)`
Controls ptrace access to another process.

**Parameters**:
- `child`: Process to be traced
- `mode`: Ptrace mode flags

**Returns**: 0 if permitted, -EPERM if denied

**Permission Logic**:
1. **Same Namespace + Subset Capabilities**: Allow if tracer has all child's permitted capabilities
2. **CAP_SYS_PTRACE**: Allow if tracer has ptrace capability in child's namespace
3. **Default**: Deny access

**Credential Types**:
- **PTRACE_MODE_FSCREDS**: Use effective capabilities
- **Default**: Use permitted capabilities

**RCU Protection**: Uses RCU read lock for credential access safety

#### `cap_ptrace_traceme(struct task_struct *parent)`
Controls whether current process can be traced by another.

**Parameters**:
- `parent`: Proposed tracer process

**Returns**: 0 if permitted, -EPERM if denied

**Permission Logic**: Similar to `cap_ptrace_access_check` but from child's perspective:
1. Check if parent has sufficient capabilities over child
2. Check if parent has `CAP_SYS_PTRACE` in child's namespace
3. Deny if neither condition met

## Security Model

### Namespace Hierarchy
The capability model respects user namespace hierarchy:
- **Parent Authority**: Capabilities in parent namespace apply to child namespaces
- **Owner Privileges**: Namespace owner has all capabilities within their namespace
- **Level Restrictions**: Cannot grant capabilities to higher-level namespaces

### Credential Context
Different credential contexts for different operations:
- **Effective Capabilities**: For most access checks
- **Permitted Capabilities**: For capability inheritance and limits
- **Filesystem Credentials**: For file-based operations

### RCU Safety
All credential access protected by RCU:
- **Read Locks**: Protect against credential structure changes
- **Current Credentials**: Safe access to current process credentials
- **Task Credentials**: Safe access to other process credentials

## Integration Points

### LSM Framework
- **Hook Implementation**: Provides implementations for LSM security hooks
- **Default Behavior**: Serves as default when no other LSM loaded
- **Composition**: Can be combined with other security modules

### Tracing System
- **Capability Traces**: Detailed tracing of capability checks
- **Security Auditing**: Integration with kernel tracing infrastructure
- **Debug Support**: Helps diagnose capability-related issues

### User Namespace System
- **Namespace Awareness**: All operations namespace-aware
- **Privilege Mapping**: Proper privilege mapping across namespace boundaries
- **Owner Rights**: Implements namespace owner privilege model

## Performance Considerations

### Fast Path Optimization
- **Common Case**: Optimized for same-namespace capability checks
- **Early Returns**: Quick exits for obvious permit/deny cases
- **RCU Usage**: Minimal locking overhead for credential access

### Memory Efficiency
- **Static Warnings**: One-time warnings prevent log spam
- **Minimal State**: No persistent state beyond credentials
- **Stack Usage**: Minimal stack footprint for capability checks

## Error Handling

### Comprehensive Coverage
- **Invalid Credentials**: Handles NULL or invalid credential pointers
- **Namespace Issues**: Proper handling of namespace hierarchy problems
- **Permission Denied**: Clear error codes for failed capability checks

### Debug Support
- **Warning Messages**: Clear messages for configuration issues
- **Trace Points**: Detailed tracing for debugging capability problems
- **Audit Integration**: Security event logging for capability usage

## Security Implications

### Privilege Escalation Prevention
- **Careful Checking**: Rigorous capability verification
- **Namespace Boundaries**: Respects namespace privilege boundaries
- **Default Deny**: Secure default behavior

### Attack Surface Reduction
- **Minimal Code**: Simple, auditable implementation
- **Clear Semantics**: Well-defined behavior reduces confusion
- **Standard Interface**: Uses standard LSM hook interface

### Compatibility
- **POSIX Compliance**: Implements POSIX.1e draft capability model
- **Legacy Support**: Maintains compatibility with older capability usage
- **Standard Behavior**: Consistent with other Unix-like systems

This implementation provides the foundational capability checking logic that underpins much of Linux's fine-grained privilege system, enabling secure privilege separation while maintaining compatibility with existing security models.