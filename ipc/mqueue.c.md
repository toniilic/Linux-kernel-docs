# mqueue.c - POSIX Message Queues Implementation

## Overview

The `mqueue.c` file implements POSIX message queues as a virtual filesystem in Linux. This provides a mechanism for inter-process communication through named message queues that support priority-based message ordering, asynchronous notification, and proper synchronization between processes.

## File Location
- **Path**: `ipc/mqueue.c`
- **License**: GPL
- **Authors**: 
  - Krzysztof Benedyczak (golbi@mat.uni.torun.pl)
  - Michal Wronski (michal.wronski@gmail.com)
  - Various contributors for spinlocks, lockless operations, and audit support

## Key Constants

### Filesystem Magic Numbers
- **MQUEUE_MAGIC**: 0x19800202 - Filesystem magic number for mqueue
- **DIRENT_SIZE**: 20 - Directory entry size
- **FILENT_SIZE**: 80 - File entry size

### Operation Types
- **SEND**: 0 - Send operation identifier
- **RECV**: 1 - Receive operation identifier

### Wait Queue States
- **STATE_NONE**: 0 - No specific state
- **STATE_READY**: 1 - Ready for operation

## Core Data Structures

### mqueue_fs_context
```c
struct mqueue_fs_context {
    struct ipc_namespace *ipc_ns;    // IPC namespace
    bool newns;                      // True if newly created namespace
};
```

Context structure for mounting the mqueue filesystem.

### posix_msg_tree_node
```c
struct posix_msg_tree_node {
    struct rb_node rb_node;          // Red-black tree node
    struct list_head msg_list;       // List of messages with same priority
    int priority;                    // Message priority level
};
```

Node in the priority-ordered red-black tree for organizing messages.

### ext_wait_queue
```c
struct ext_wait_queue {
    struct task_struct *task;        // Waiting task
    struct list_head list;           // Wait queue list node
    struct msg_msg *msg;             // Pointer to loaded message
    int state;                       // Current state (STATE_*)
};
```

Extended wait queue for tasks waiting on message queue operations.

### mqueue_inode_info
```c
struct mqueue_inode_info {
    spinlock_t lock;                 // Queue synchronization lock
    struct inode vfs_inode;          // VFS inode (must be second)
    wait_queue_head_t wait_q;        // Standard wait queue
    
    struct rb_root msg_tree;         // Red-black tree of messages
    struct rb_node *msg_tree_rightmost; // Rightmost node cache
    struct posix_msg_tree_node *node_cache; // Cached tree node
    struct mq_attr attr;             // Queue attributes
    
    struct sigevent notify;          // Notification event structure
    struct pid *notify_owner;        // Owner PID for notifications
    u32 notify_self_exec_id;         // Exec ID for notification validation
    struct user_namespace *notify_user_ns; // User namespace for notifications
    struct ucounts *ucounts;         // User accounting structure
    struct sock *notify_sock;        // Notification socket
    struct sk_buff *notify_cookie;   // Notification cookie
    
    struct ext_wait_queue e_wait_q[2]; // Wait queues for send/recv
    
    unsigned long qsize;             // Total queue size in memory
};
```

Core structure representing a message queue, embedded in the inode.

## Memory Barriers and Synchronization

### Critical Synchronization Issues

The code contains extensive documentation about memory barrier requirements to prevent race conditions:

#### 1. Task Lifecycle Races
**Problem**: Task could exit before reference counting completes
**Solution**: Use `wake_q_add_safe()` and acquire task reference before state change

#### 2. Stale Data Access
**Problem**: Woken task might read stale message data
**Solution**: Use proper release/acquire memory barriers with `smp_store_release()` and `smp_acquire__after_ctrl_dep()`

#### 3. Lock-Free State Checking
**Problem**: Exit paths check state without locks
**Solution**: Careful ordering with proper memory barriers

### Locking Protocol
- **Primary Lock**: `info->lock` synchronizes all queue access
- **Exception 1**: Wake-up operations use wake_q framework after lock release
- **Exception 2**: Exit paths check `ext_wait_queue->state` lock-free

## Core Functions

### Namespace Management

#### `MQUEUE_I(struct inode *inode)`
Converts inode to mqueue_inode_info structure.
- **Parameters**: VFS inode
- **Returns**: Embedded mqueue_inode_info structure
- **Implementation**: Uses `container_of()` macro

#### `__get_ns_from_inode(struct inode *inode)`
Retrieves IPC namespace from inode (requires mq_lock).
- **Parameters**: VFS inode
- **Returns**: IPC namespace with reference
- **Locking**: Must be called with `mq_lock` held

#### `get_ns_from_inode(struct inode *inode)`
Thread-safe version of namespace retrieval.
- **Parameters**: VFS inode  
- **Returns**: IPC namespace with reference
- **Thread Safety**: Acquires and releases `mq_lock`

### Message Management

#### `msg_insert(struct msg_msg *msg, struct mqueue_inode_info *info)`
Inserts a message into the priority-ordered message tree.
- **Parameters**: Message structure and queue info
- **Returns**: 0 on success, error code on failure
- **Algorithm**: 
  1. Traverses red-black tree to find insertion point
  2. Creates new tree node if priority doesn't exist
  3. Adds message to priority-specific list
  4. Maintains rightmost node cache for efficiency

### Data Structure Implementation

#### Red-Black Tree Organization
- **Key**: Message priority (higher priority = lower numerical value)
- **Structure**: Each priority level has a tree node containing a list of messages
- **Optimization**: Rightmost node cached for fast access to lowest priority
- **Ordering**: Messages with same priority maintained in FIFO order

#### Wait Queue Management
- **Dual Queues**: Separate wait queues for send and receive operations
- **Extended Structure**: Custom wait queue with message pointer and state
- **Lock-Free Access**: State checking optimized for fast exit paths

## Filesystem Integration

### VFS Operations
- **File System Type**: `mqueue_fs_type` - Registers mqueue as VFS
- **Inode Operations**: `mqueue_dir_inode_operations` - Directory operations
- **File Operations**: `mqueue_file_operations` - File-level operations
- **Super Operations**: `mqueue_super_ops` - Superblock operations
- **Context Operations**: `mqueue_fs_context_ops` - Mount context handling

### Memory Management
- **Inode Cache**: `mqueue_inode_cachep` - SLAB cache for mqueue inodes
- **Message Storage**: Messages stored in kernel memory with size accounting
- **User Accounting**: `ucounts` structure tracks per-user resource usage

## Notification System

### Asynchronous Notifications
- **Signal Notifications**: Can send signals when messages arrive
- **Socket Notifications**: Can send netlink messages for notifications
- **Thread Safety**: Notification owner validated using exec IDs
- **Cleanup**: `remove_notification()` handles cleanup on queue destruction

### Notification Components
- **Event Structure**: `sigevent` defines notification method
- **Owner Tracking**: PID and exec ID prevent stale notifications
- **Network Integration**: Socket-based notifications for advanced use cases

## Performance Optimizations

### Message Tree Efficiency
- **Red-Black Tree**: O(log n) insertion and deletion
- **Priority Caching**: Rightmost node cached for fast low-priority access
- **Node Reuse**: Tree node cache reduces allocation overhead

### Lock-Free Fast Paths
- **State Checking**: Exit paths avoid lock acquisition when possible
- **Memory Barriers**: Proper ordering without excessive locking
- **Wake Queue**: Batched waking reduces lock contention

### Memory Efficiency
- **Embedded Inodes**: mqueue_inode_info embedded in VFS inode structure
- **SLAB Caching**: Custom cache for mqueue inodes
- **Size Tracking**: Accurate memory usage accounting

## Security and Accounting

### User Limits
- **Per-User Accounting**: `ucounts` tracks resource usage per user
- **Namespace Isolation**: IPC namespaces provide isolation
- **Permission Checking**: Standard POSIX permission model

### Resource Management
- **Queue Size Limits**: Maximum queue size enforced
- **Message Count Limits**: Maximum number of messages enforced
- **Memory Accounting**: Total memory usage tracked and limited

## Thread Safety Guarantees

### Synchronization Model
- **Single Lock**: Primary spinlock protects all queue operations
- **Wake Queue Framework**: Safe task waking without holding locks
- **Memory Barriers**: Proper ordering for lock-free state access

### Race Prevention
- **Reference Counting**: Prevents use-after-free in concurrent scenarios
- **State Validation**: Exec ID validation prevents stale notifications
- **Ordered Operations**: Careful ordering prevents data races

## Error Handling

### Resource Exhaustion
- **Memory Allocation**: Graceful handling of allocation failures
- **Queue Limits**: Proper error returns when limits exceeded
- **Cleanup Paths**: Comprehensive cleanup on error conditions

### Signal Safety
- **Interruptible Waits**: Operations can be interrupted by signals
- **Signal Delivery**: Proper signal handling during blocking operations
- **Restart Logic**: Correct handling of interrupted system calls

## Integration with POSIX Standards

### POSIX Compliance
- **Message Priorities**: Support for priority-ordered message delivery
- **Blocking/Non-blocking**: Support for both blocking and non-blocking operations
- **Attributes**: Complete implementation of mq_attr structure
- **Notifications**: Full support for mq_notify() functionality

### System Call Interface
- **mq_open()**: Queue creation and opening
- **mq_send()/mq_receive()**: Message operations
- **mq_notify()**: Asynchronous notification setup
- **mq_getattr()/mq_setattr()**: Attribute management

This implementation provides a robust, efficient, and POSIX-compliant message queue system that integrates seamlessly with the Linux VFS while providing the synchronization and performance characteristics required for inter-process communication.