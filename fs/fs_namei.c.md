# fs/namei.c - Pathname Resolution and Namespace Navigation

## Overview

This file implements the core pathname resolution (namei) subsystem in the Linux kernel. Originally written by Linus Torvalds and significantly rewritten by T. Schoebel-Theuer and Al Viro, it handles the complex task of converting pathnames to filesystem objects while dealing with symlinks, mount points, permissions, and security constraints.

## Historical Evolution

### Major Rewrites
- **1991-1992**: Original implementation by Linus Torvalds
- **February 1997**: Complete rewrite by T. Schoebel-Theuer for iterative symlink resolution
- **February-April 2000**: Al Viro's rewrite for new namespace architecture
- **Ongoing**: Continuous optimization for RCU-walk and performance

### Key Improvements Over Time
- **Recursive → Iterative**: Symlink resolution changed from recursive to iterative
- **RCU-walk**: Lock-free path walking for performance
- **Namespace Support**: Container and mount namespace integration
- **Security Hardening**: Enhanced security checks and access controls

## Core Concepts

### Pathname Resolution Process
1. **Input Processing**: Parse and validate pathname string
2. **Path Walking**: Navigate through directory hierarchy
3. **Symlink Resolution**: Handle symbolic links iteratively
4. **Mount Traversal**: Cross filesystem boundaries
5. **Permission Checking**: Verify access at each step
6. **Final Resolution**: Return target inode/dentry

### Resolution Modes

#### RCU-walk Mode
- **Lock-Free**: No locks taken during path walk
- **Fast Path**: Optimized for common cases
- **Fallback**: Falls back to ref-walk on conflicts
- **Scalability**: Excellent SMP scalability

#### Ref-walk Mode
- **Reference Counting**: Takes references on dentries
- **Reliable**: Always succeeds if path exists
- **Slower**: Higher overhead but guaranteed progress
- **Complex Cases**: Handles all edge cases

## Key Data Structures

### `struct nameidata` - Path Walking State
```c
struct nameidata {
    struct path path;           /* Current position */
    struct qstr last;          /* Last component being resolved */
    struct path root;          /* Root directory for resolution */
    struct inode *inode;       /* Current inode */
    unsigned int flags, state; /* Control flags and state */
    unsigned seq, next_seq;    /* RCU sequence numbers */
    int last_type;            /* Type of last component */
    unsigned depth;           /* Symlink recursion depth */
    int total_link_count;     /* Total symlinks followed */
    struct saved *stack;      /* Symlink resolution stack */
    struct filename *name;    /* Original filename */
    const char *pathname;     /* Current pathname position */
};
```

**Key Fields**:
- **path**: Current directory and vfsmount
- **last**: Last component name for resolution
- **root**: Root directory (for chroot, containers)
- **flags**: Control behavior (RCU, create, etc.)
- **seq**: RCU sequence numbers for consistency
- **stack**: Stack for nested symlink resolution

### `struct filename` - Pathname Container
```c
struct filename {
    const char *name;        /* Pathname string */
    const __user char *uptr; /* Original user pointer */
    atomic_t refcnt;        /* Reference count */
    struct audit_names *aname; /* Audit information */
    const char iname[];     /* Inline name storage */
};
```

### Path Walking Flags
- **LOOKUP_FOLLOW**: Follow final symlink
- **LOOKUP_DIRECTORY**: Expect directory result
- **LOOKUP_RCU**: Use RCU-walk mode
- **LOOKUP_CREATE**: Creating new file
- **LOOKUP_EXCL**: Exclusive creation
- **LOOKUP_RENAME_TARGET**: Rename operation target

## Core Functions

### `filename_lookup()` - Main Entry Point
```c
static int filename_lookup(int dfd, struct filename *filename,
                          unsigned flags, struct path *path,
                          struct path *root)
```

**Purpose**: Primary interface for pathname resolution

**Process**:
1. **Setup**: Initialize nameidata structure
2. **RCU Attempt**: Try RCU-walk first for performance
3. **Ref-walk Fallback**: Use ref-walk if RCU fails
4. **Result**: Return resolved path

**Optimization Strategy**:
- **RCU First**: Attempt lock-free resolution
- **Graceful Fallback**: Switch to ref-walk when needed
- **Audit Integration**: Handle security auditing

### `path_lookupat()` - Core Resolution Logic
```c
static int path_lookupat(struct nameidata *nd, unsigned flags, struct path *path)
```

**Resolution Process**:
1. **Initialization**: Set up initial path state
2. **Component Walk**: Process each pathname component
3. **Symlink Handling**: Resolve symbolic links
4. **Mount Traversal**: Handle filesystem boundaries
5. **Final Validation**: Check final result

### `walk_component()` - Single Component Resolution
```c
static int walk_component(struct nameidata *nd, int flags)
```

**Component Processing**:
1. **Name Lookup**: Find component in current directory
2. **Permission Check**: Verify search permission
3. **Type Validation**: Check if component type matches expectation
4. **Mount Handling**: Handle mount point crossing
5. **Symlink Detection**: Identify and queue symlinks

### Symlink Resolution System

#### `pick_link()` - Symlink Queuing
```c
static const char *pick_link(struct nameidata *nd, struct path *link,
                            struct inode *inode, unsigned seq, int flags)
```

**Symlink Management**:
- **Stack Management**: Push symlink onto resolution stack
- **Loop Detection**: Prevent infinite symlink loops
- **Depth Limiting**: Enforce maximum symlink depth
- **RCU Handling**: Handle RCU considerations for symlinks

#### `step_into()` - Path Component Transition
```c
static int step_into(struct nameidata *nd, int flags,
                    struct dentry *dentry, struct inode *inode, unsigned seq)
```

**State Transition**:
- **Mount Crossing**: Handle mount point traversal
- **Directory Entry**: Update current position
- **RCU Consistency**: Maintain RCU invariants
- **Error Handling**: Handle transition failures

### Permission and Security

#### `inode_permission()` - Access Control
```c
int inode_permission(struct mnt_idmap *idmap, struct inode *inode, int mask)
```

**Permission Checking**:
1. **Basic Checks**: File mode bits and ownership
2. **ACL Processing**: POSIX ACLs if present
3. **LSM Integration**: Security module hooks
4. **Special Cases**: Handle special files and conditions

**Access Types**:
- **MAY_READ**: Read permission
- **MAY_WRITE**: Write permission
- **MAY_EXEC**: Execute/search permission
- **MAY_APPEND**: Append-only access

#### Security Features

##### Protected Symlinks
```c
static int may_follow_link(struct nameidata *nd, const struct inode *inode)
```

**Symlink Security**:
- **Ownership Checks**: Verify symlink ownership
- **World-Writable Protection**: Restrict in world-writable directories
- **Sticky Bit**: Honor sticky bit semantics
- **Configuration**: Controlled by sysctl settings

##### Protected Hardlinks
```c
int may_linkat(struct mnt_idmap *idmap, const struct path *link)
```

**Hardlink Security**:
- **Ownership Verification**: Ensure proper ownership
- **Mode Restrictions**: Check file permissions
- **Process Context**: Verify process capabilities
- **Prevention**: Prevent privilege escalation

### Mount Point Handling

#### `__follow_mount_rcu()` - RCU Mount Traversal
```c
static int __follow_mount_rcu(struct nameidata *nd, struct path *path,
                             struct inode **inode, unsigned *seqp)
```

**Mount Crossing**:
- **Automount Detection**: Handle automount points
- **Mount Stack**: Handle multiple mounted filesystems
- **RCU Safety**: Maintain RCU consistency during traversal
- **Sequence Validation**: Ensure mount state consistency

#### `follow_mount()` - Reference-Based Mount Traversal
```c
static int follow_mount(struct path *path)
```

**Mount Resolution**:
- **Reference Taking**: Take references on mount structures
- **Automount Triggering**: Trigger automounts when needed
- **Error Handling**: Handle mount failures gracefully
- **Cleanup**: Proper reference cleanup

### RCU-walk Implementation

#### `unlazy_walk()` - RCU to Ref-walk Transition
```c
static int unlazy_walk(struct nameidata *nd)
```

**Mode Transition**:
1. **Validation**: Verify RCU-walk state is still valid
2. **Reference Taking**: Take references on current path
3. **Cleanup**: Clean up RCU state
4. **Continuation**: Enable continued ref-walk

**Transition Triggers**:
- **Sequence Mismatch**: RCU sequence changed
- **Complex Operations**: Operations requiring locks
- **Error Conditions**: When RCU-walk cannot proceed

#### `complete_walk()` - Path Validation
```c
static int complete_walk(struct nameidata *nd)
```

**Final Validation**:
- **RCU Completion**: Finish RCU-walk if still active
- **Reference Validation**: Verify all references valid
- **Path Connectivity**: Ensure path is still connected
- **Security Validation**: Final security checks

## System Call Implementations

### Pathname-Based System Calls

#### `do_sys_openat2()` → `filename_lookup()`
- **File Opening**: Resolve path for open operations
- **Creation Handling**: Handle O_CREAT and related flags
- **Mode Validation**: Check file types and permissions

#### `do_mknodat()` → `filename_parentat()`
- **Parent Resolution**: Find parent directory
- **Name Validation**: Validate final component name
- **Creation Permission**: Check creation permissions

#### `do_sys_name_to_handle_at()` → `filename_lookup()`
- **Handle Generation**: Convert pathname to file handle
- **Export Support**: Support for NFS exports
- **Permission Checking**: Verify access permissions

### Path-Based Operations

#### `vfs_path_lookup()`
```c
int vfs_path_lookup(struct dentry *dentry, struct vfsmount *mnt,
                   const char *name, unsigned int flags, struct path *path)
```

**Kernel Path Resolution**:
- **Kernel Interface**: Direct kernel path lookup
- **No User Copy**: Avoid user space string copying
- **Context Control**: Explicit context specification

## Performance Optimizations

### RCU-walk Benefits
- **Lock-Free**: No locks during common path resolution
- **Scalability**: Excellent SMP performance
- **Cache Efficiency**: Better CPU cache utilization
- **Reduced Contention**: Eliminates lock contention

### Fast Path Optimizations
- **Inline Caching**: Cache recent lookups
- **Sequence Numbers**: Efficient consistency checking
- **Stack Allocation**: Use stack for small symlink chains
- **Early Termination**: Stop resolution when possible

### Memory Management
- **Embedded Storage**: Small filename storage inline
- **Reference Counting**: Efficient memory management
- **RCU Protection**: Safe concurrent access
- **Stack Reuse**: Reuse symlink resolution stack

## Security Framework Integration

### Linux Security Modules (LSM)
- **Path-Based Controls**: Control access based on paths
- **Context Preservation**: Maintain security contexts
- **Hook Integration**: Integrate with LSM hooks
- **Policy Enforcement**: Enforce security policies

### Audit Subsystem
- **Path Recording**: Record accessed paths
- **Operation Tracking**: Track filesystem operations
- **Security Events**: Generate security audit events
- **Compliance**: Support compliance requirements

### Access Control Lists (ACL)
- **POSIX ACLs**: Support POSIX access control lists
- **Extended Permissions**: Beyond traditional Unix permissions
- **Inheritance**: Handle ACL inheritance
- **Performance**: Efficient ACL checking

## Namespace Integration

### Mount Namespaces
- **Isolated Views**: Different mount views per namespace
- **Bind Mounts**: Handle bind mount semantics
- **Pivot Root**: Support pivot_root operation
- **Propagation**: Handle mount propagation

### Container Support
- **Root Isolation**: Isolated root directories
- **Path Translation**: Translate paths across namespaces
- **Security Boundaries**: Maintain security isolation
- **Resource Limits**: Respect container resource limits

## Error Handling and Edge Cases

### Common Error Conditions
- **ENOENT**: Path component doesn't exist
- **EACCES**: Permission denied
- **ELOOP**: Too many symbolic links
- **ENAMETOOLONG**: Pathname too long
- **ENOTDIR**: Component is not a directory

### Race Condition Handling
- **RCU Consistency**: Handle concurrent modifications
- **Reference Validation**: Verify references remain valid
- **Sequence Checking**: Detect concurrent changes
- **Graceful Fallback**: Fall back to slower but safe methods

### Resource Management
- **Memory Allocation**: Handle allocation failures
- **Reference Leaks**: Prevent reference leaks
- **Stack Overflow**: Prevent stack overflow in deep paths
- **Cleanup Paths**: Ensure proper cleanup on errors

## Debugging and Monitoring

### Tracing Support
- **ftrace Integration**: Support for function tracing
- **Path Tracking**: Track path resolution steps
- **Performance Analysis**: Analyze resolution performance
- **Debugging Aid**: Help debug path-related issues

### Statistics and Metrics
- **Performance Counters**: Track resolution performance
- **Cache Hit Rates**: Monitor cache effectiveness
- **Error Rates**: Track error frequencies
- **Security Events**: Monitor security-related events

This implementation represents one of the most sophisticated pathname resolution systems in any operating system, balancing performance, security, and correctness while supporting modern features like containers, namespaces, and high-performance concurrent access patterns.