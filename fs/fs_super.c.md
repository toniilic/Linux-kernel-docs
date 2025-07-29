# Linux Kernel Superblock Management (`fs/super.c`)

## Overview

The `fs/super.c` file implements Linux's superblock management system, a critical component that manages filesystem superblocks throughout their lifecycle. This subsystem handles superblock allocation, initialization, mounting, unmounting, reference counting, freeze/thaw operations, and cleanup. The superblock serves as the primary in-memory representation of a mounted filesystem, containing essential metadata and coordinating all filesystem operations.

## Core Architecture

### 1. Superblock State Management

**Superblock States** - Lines 150, 679:
```c
#define SUPER_WAKE_FLAGS (SB_BORN | SB_DYING | SB_DEAD)
static void super_wake(struct super_block *sb, unsigned int flag) {
    smp_store_release(&sb->s_flags, sb->s_flags | flag);
    smp_mb();
    wake_up_var(&sb->s_flags);
}
```

**State Transitions**:
- **SB_BORN**: Superblock fully initialized and ready for use
- **SB_DYING**: Superblock being shut down, new operations blocked
- **SB_DEAD**: Superblock completely destroyed, safe to free
- **SB_ACTIVE**: Superblock has active references and operations

### 2. Superblock Locking

**Reader/Writer Locking** - Lines 54-83:
```c
static inline void __super_lock(struct super_block *sb, bool excl) {
    if (excl)
        down_write(&sb->s_umount);
    else
        down_read(&sb->s_umount);
}
```

**Lock Hierarchy**:
- **s_umount**: Primary superblock read/write semaphore
- **sb_lock**: Global spinlock protecting superblock lists
- **s_sync_lock**: Synchronization for filesystem operations
- **s_writers**: Per-CPU reader/writer semaphores for freeze operations

### 3. Global Superblock Management

**Global Lists** - Lines 45-46:
```c
static LIST_HEAD(super_blocks);          // All superblocks
static DEFINE_SPINLOCK(sb_lock);          // Protects global lists
```

## Superblock Lifecycle Management

### 1. Superblock Allocation

**`alloc_super()`** - Lines 316-398:
- **Memory Allocation**: Allocates and zero-initializes superblock structure
- **Lock Initialization**: Sets up all locking primitives (s_umount, sync_lock, etc.)
- **Security Integration**: Initializes security contexts via LSM hooks
- **Freeze Infrastructure**: Sets up per-CPU reader/writer semaphores for freeze/thaw
- **Shrinker Setup**: Configures memory shrinker for cache management
- **LRU Initialization**: Sets up dentry and inode LRU caches

**Initialization Sequence**:
1. **Memory Allocation**: kzalloc superblock structure
2. **Basic Setup**: Initialize lists, locks, and basic fields
3. **Security Setup**: Call security_sb_alloc() for LSM initialization
4. **Freeze Setup**: Initialize per-CPU rwsems for freeze levels
5. **Cache Setup**: Initialize LRU lists and shrinker registration
6. **Default Values**: Set reasonable defaults for time granularity, etc.

### 2. Superblock Discovery and Reuse

**`sget_fc()` / `sget()`** - Lines 731-867:
- **Filesystem Type Lookup**: Search existing superblocks by filesystem type
- **Test Function**: Use custom test function to match superblocks
- **User Namespace Validation**: Ensure namespace compatibility
- **Reference Management**: Handle active/temporary reference transitions
- **Retry Logic**: Handle race conditions during allocation/lookup

**Lookup Algorithm**:
```
1. Acquire sb_lock
2. Search fs_type->fs_supers list using test function
3. If found and compatible:
   - grab_super() to get active reference
   - Return existing superblock
4. If not found:
   - Release lock, allocate new superblock
   - Retry search (handle allocation races)
   - Initialize and publish new superblock
```

### 3. Superblock Mounting

**`vfs_get_tree()`** - Lines 1793-1846:
- **Filesystem Invocation**: Calls filesystem's get_tree() method
- **Root Validation**: Ensures filesystem set fc->root properly
- **State Transition**: Marks superblock as SB_BORN when ready
- **Security Setup**: Applies security mount options
- **Validation**: Performs sanity checks on filesystem parameters

**Mount Process**:
1. **Get Tree**: Call filesystem-specific get_tree() operation
2. **Validate Root**: Ensure fc->root is properly set
3. **Security Setup**: Apply LSM mount options
4. **State Update**: Mark superblock as SB_BORN
5. **Validation**: Check s_maxbytes and other limits

## Reference Counting and Lifetime

### 1. Reference Count Management

**Reference Types**:
- **s_count**: Temporary references (protected by sb_lock)
- **s_active**: Active references (atomic, indicates usable state)
- **Active State**: When s_active > 0, superblock is fully functional

**`grab_super()`** - Lines 525-542:
- **State Validation**: Ensures superblock is in correct state for use
- **Active Reference**: Converts temporary reference to active reference
- **Lock Coordination**: Manages s_umount locking during transition
- **Race Handling**: Handles concurrent shutdown scenarios

### 2. Superblock Deactivation

**`deactivate_locked_super()`** - Lines 469-491:
- **Active Reference Drop**: Decrements s_active atomically
- **Shutdown Trigger**: Calls filesystem's kill_sb() when s_active reaches 0
- **Resource Cleanup**: Destroys shrinker and LRU caches
- **Notification**: Signals waiters about superblock death

**Deactivation Process**:
1. **Reference Check**: atomic_dec_and_test(&s->s_active)
2. **Shrinker Cleanup**: Free memory shrinker
3. **Filesystem Shutdown**: Call fs->kill_sb(s)
4. **Death Notification**: kill_super_notify() signals SB_DEAD
5. **LRU Cleanup**: Destroy dentry and inode LRU caches

### 3. Memory Management Integration

**`super_cache_scan/count()`** - Lines 178-272:
- **Memory Pressure Response**: Integrates with kernel memory reclaim
- **Proportional Scanning**: Balances between dcache, icache, and fs caches
- **Lock-free Counting**: Provides cache counts without heavy locking
- **Filesystem Integration**: Calls filesystem-specific cache operations

**Cache Management Strategy**:
- **Dentry Cache**: Prune dcache first (pinned by icache)
- **Inode Cache**: Prune icache after dcache
- **Filesystem Cache**: Call filesystem-specific free_cached_objects()
- **Proportional Distribution**: Split scan between cache types based on size

## Freeze/Thaw Operations

### 1. Filesystem Freezing

**`freeze_super()`** - Lines 2120-2209:
- **Multi-level Freezing**: Implements staged freeze process
- **Writer Coordination**: Uses per-CPU rwsems to coordinate with writers
- **Holder Tracking**: Distinguishes between kernel and userspace freezes
- **Nested Freezing**: Supports multiple concurrent freeze operations
- **State Progression**: Manages freeze state transitions

**Freeze Levels** - Lines 48-52:
```c
static char *sb_writers_name[SB_FREEZE_LEVELS] = {
    "sb_writers",      // SB_FREEZE_WRITE: Block new writes
    "sb_pagefaults",   // SB_FREEZE_PAGEFAULT: Block page faults
    "sb_internal",     // SB_FREEZE_FS: Block internal fs operations
};
```

**Freeze State Machine**:
```
SB_UNFROZEN → SB_FREEZE_WRITE → SB_FREEZE_PAGEFAULT → SB_FREEZE_FS → SB_FREEZE_COMPLETE
```

### 2. Freeze Process Implementation

**Staged Freezing Process**:
1. **SB_FREEZE_WRITE**: Block new write operations, wait for existing writes
2. **SB_FREEZE_PAGEFAULT**: Block page faults, continue waiting
3. **Sync Filesystem**: Call sync_filesystem() to flush all dirty data
4. **SB_FREEZE_FS**: Block internal filesystem operations
5. **Filesystem Freeze**: Call sb->s_op->freeze_fs() for fs-specific freezing
6. **SB_FREEZE_COMPLETE**: Mark freeze complete, release s_umount lock

**Writer Coordination** - Lines 1898-1901:
```c
static void sb_wait_write(struct super_block *sb, int level) {
    percpu_down_write(sb->s_writers.rw_sem + level-1);
}
```

### 3. Thaw Operations

**`thaw_super_locked()`** - Lines 2217-2265:
- **Permission Validation**: Checks if caller can unfreeze filesystem
- **Reference Counting**: Manages freeze reference counts per holder type
- **State Restoration**: Reverses freeze process when all references dropped
- **Filesystem Integration**: Calls filesystem's unfreeze_fs() operation
- **Lock Release**: Releases all acquired freeze locks

**Thaw Process**:
1. **Validation**: Check if caller is authorized to thaw
2. **Reference Management**: Decrement freeze reference count
3. **State Check**: If still frozen by others, just return
4. **Filesystem Thaw**: Call sb->s_op->unfreeze_fs() if available
5. **Lock Release**: Release all freeze-level rwsem locks
6. **State Update**: Mark as SB_UNFROZEN and wake waiters

## Block Device Integration

### 1. Block Device Support

**`setup_bdev_super()`** - Lines 1592-1641:
- **Device Opening**: Opens block device with appropriate permissions
- **Read-only Validation**: Ensures write operations allowed when needed
- **Freeze Detection**: Prevents mounting frozen block devices
- **BDI Integration**: Sets up backing device info for I/O operations
- **Block Size**: Configures superblock block size from device

**Block Device Operations** - Lines 1584-1590:
```c
const struct blk_holder_ops fs_holder_ops = {
    .mark_dead    = fs_bdev_mark_dead,    // Handle device removal
    .sync         = fs_bdev_sync,         // Sync filesystem
    .freeze       = fs_bdev_freeze,       // Freeze from block layer
    .thaw         = fs_bdev_thaw,         // Thaw from block layer
};
```

### 2. Device State Management

**`fs_bdev_mark_dead()`** - Lines 1454-1470:
- **Graceful Shutdown**: Syncs filesystem before marking device dead
- **Cache Cleanup**: Shrinks dcache and evicts inodes
- **Filesystem Notification**: Calls filesystem's shutdown operation
- **State Coordination**: Manages superblock locking during shutdown

**Device Integration Features**:
- **Hot Removal**: Handles surprise device removal gracefully
- **Sync Operations**: Coordinates filesystem sync with block layer
- **Freeze Integration**: Allows block layer to freeze/thaw filesystems
- **Error Handling**: Manages I/O errors and device failures

## Mount Helpers and Utilities

### 1. Anonymous Device Support

**`get_anon_bdev()`** - Lines 1247-1265:
- **Virtual Device Creation**: Allocates virtual block device numbers
- **FSID Generation**: Ensures non-zero FSID for userspace compatibility
- **Resource Management**: Uses IDA for efficient device number allocation
- **Range Management**: Respects MINORBITS limitations

**Anonymous Device Features**:
- **No Physical Device**: For filesystems without block devices (tmpfs, proc)
- **Unique Identification**: Provides unique device numbers for stat()
- **Resource Cleanup**: Automatic cleanup via free_anon_bdev()

### 2. Mount Type Helpers

**Mount Helper Functions**:
- **`get_tree_nodev()`**: Mount without block device
- **`get_tree_single()`**: Singleton filesystem mount
- **`get_tree_keyed()`**: Keyed filesystem mount (same key = same sb)
- **`get_tree_bdev()`**: Block device based mount

**Mount Patterns**:
```c
int get_tree_single(struct fs_context *fc,
    int (*fill_super)(struct super_block *, struct fs_context *)) {
    return vfs_get_super(fc, test_single_super, fill_super);
}
```

## Emergency Operations

### 1. Emergency Remount

**`emergency_remount()`** - Lines 1131-1140:
- **System Recovery**: Remounts all filesystems read-only during emergencies
- **Work Queue**: Uses work queue to avoid blocking emergency context
- **Reverse Order**: Processes superblocks in reverse order for safety
- **Error Handling**: Continues operation even if individual remounts fail

**Emergency Scenarios**:
- **System Panic**: During kernel panic for data protection
- **Critical Errors**: When system integrity is compromised
- **Power Loss**: Before emergency shutdown
- **Memory Pressure**: During severe memory shortages

### 2. Emergency Thaw

**`emergency_thaw_all()`** - Lines 1163-1172:
- **SysRq Integration**: Triggered via Magic SysRq key
- **Forced Thaw**: Thaws all frozen filesystems unconditionally
- **System Recovery**: Restores system to operational state
- **Block Device Thaw**: Also thaws block devices if needed

## Advanced Features

### 1. Namespace Integration

**User Namespace Support**:
- **Namespace Validation**: Ensures superblock/mount namespace compatibility
- **Permission Checking**: Validates capability in appropriate namespace
- **Security Boundaries**: Maintains security isolation between namespaces
- **FS_USERNS_MOUNT**: Flag indicating filesystem supports user namespaces

### 2. Security Integration

**LSM Coordination**:
- **security_sb_alloc()**: Initialize security context during allocation
- **security_sb_remount()**: Validate remount security parameters
- **security_sb_set_mnt_opts()**: Apply security mount options
- **security_sb_delete()**: Clean up security state during shutdown

### 3. Shrinker Integration

**Memory Reclaim**:
- **NUMA Awareness**: Shrinker respects NUMA topology
- **Memcg Integration**: Works with memory cgroups
- **Proportional Scanning**: Balances cache pressure across cache types
- **Deadlock Avoidance**: Prevents recursion into same filesystem

## Error Handling and Recovery

### 1. Graceful Degradation

**Failure Scenarios**:
- **Allocation Failures**: Handle memory allocation failures gracefully
- **Device Errors**: Manage block device I/O errors
- **Filesystem Errors**: Coordinate with filesystem error handling
- **Resource Exhaustion**: Handle resource limit situations

### 2. Consistency Maintenance

**State Consistency**:
- **Reference Counting**: Ensures proper resource cleanup
- **Lock Ordering**: Maintains consistent lock ordering to prevent deadlocks
- **State Transitions**: Validates state transitions for correctness
- **Race Prevention**: Uses memory barriers and atomic operations

## Performance Optimizations

### 1. Lock Granularity

**Fine-grained Locking**:
- **Per-superblock Locks**: Minimizes contention between filesystems
- **RCU Protection**: Uses RCU for some read-heavy operations
- **Per-CPU Structures**: Freeze/thaw uses per-CPU rwsems for scalability
- **Lock-free Operations**: Cache counting avoids locks where possible

### 2. Memory Efficiency

**Resource Management**:
- **Object Reuse**: Reuses superblocks when possible via sget()
- **Lazy Allocation**: Allocates resources only when needed
- **Efficient Cleanup**: Uses RCU and work queues for cleanup
- **Cache Integration**: Integrates with kernel memory management

## Integration Points

### 1. VFS Integration

**Virtual File System**:
- **Mount Infrastructure**: Provides core mounting infrastructure
- **Dentry/Inode Management**: Manages dentry and inode caches
- **Security Framework**: Integrates with LSM framework
- **Namespace Support**: Works with mount and user namespaces

### 2. Block Layer Integration

**Block Device Coordination**:
- **I/O Operations**: Coordinates I/O with block layer
- **Device Management**: Handles device lifecycle events
- **Freeze/Thaw**: Supports block layer freeze/thaw operations
- **Error Propagation**: Handles block device errors

### 3. Memory Management

**MM Subsystem Integration**:
- **Shrinker Registration**: Participates in memory reclaim
- **Page Cache**: Manages page cache pressure
- **Memory Accounting**: Accounts for superblock memory usage
- **NUMA Awareness**: Respects NUMA topology for allocations

The superblock management system represents the core of Linux filesystem infrastructure, providing a robust foundation for all filesystem operations while maintaining compatibility, performance, and reliability across diverse filesystem types and usage scenarios.