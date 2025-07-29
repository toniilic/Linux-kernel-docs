# Linux Kernel File Locking System (`fs/locks.c`)

## Overview

The `fs/locks.c` file implements Linux's comprehensive file locking system, managing four distinct types of file locks: BSD locks (flock), POSIX locks (fcntl), open file description locks (OFD), and leases. This subsystem provides the foundation for coordinating access to files across processes and threads, implementing complex locking protocols including deadlock detection, lock tree management, and lease breaking mechanisms. The system ensures data consistency while supporting both advisory and mandatory locking semantics.

## Core Architecture

### 1. Lock Type Classification

**Lock Categories** - Lines 5-8:
```c
/*
 * We implement four types of file locks: BSD locks, posix locks, open
 * file description locks, and leases.
 */
```

**Lock Type Features**:
- **BSD Locks (FLOCK)**: Process-level locks, simple exclusive/shared semantics
- **POSIX Locks**: Thread-level locks with byte-range support and inheritance
- **OFD Locks**: Open file description locks, independent of process relationships
- **Leases**: Opportunistic locks for cache coherency and delegation

### 2. Lock Tree Management

**Tree Properties** - Lines 11-50:
```c
/*
 * Locking conflicts and dependencies:
 * - the root of a tree may be an applied or waiting lock
 * - every other node in the tree is a waiting lock that
 *   conflicts with every ancestor of that node
 */
```

**Tree Operations**:
- **Leaf Addition**: New conflicting locks become tree leaves
- **Root Removal**: Creates new singleton trees from children
- **Lock Application**: Promotes waiting locks to applied status
- **Range Upgrades**: Extends or promotes lock ranges

### 3. Global Lock Management

**Per-CPU Lock Lists** - Lines 135-140:
```c
struct file_lock_list_struct {
    spinlock_t        lock;
    struct hlist_head hlist;
};
static DEFINE_PER_CPU(struct file_lock_list_struct, file_lock_list);
DEFINE_STATIC_PERCPU_RWSEM(file_rwsem);
```

**Global Coordination**:
- **Per-CPU Lists**: Reduces contention in /proc/locks display
- **Global Serialization**: Uses file_rwsem for cross-CPU operations
- **Hash Tables**: Blocked lock hash for deadlock detection
- **RCU Protection**: Safe concurrent access to lock structures

## Lock Context Management

### 1. File Lock Context

**Context Structure** - Lines 176-206:
```c
static struct file_lock_context *
locks_get_lock_context(struct inode *inode, int type) {
    struct file_lock_context *ctx;
    
    ctx = locks_inode_context(inode);
    if (likely(ctx) || type == F_UNLCK)
        goto out;
        
    ctx = kmem_cache_alloc(flctx_cache, GFP_KERNEL);
    if (!ctx)
        goto out;
        
    spin_lock_init(&ctx->flc_lock);
    INIT_LIST_HEAD(&ctx->flc_flock);
    INIT_LIST_HEAD(&ctx->flc_posix);
    INIT_LIST_HEAD(&ctx->flc_lease);
}
```

**Context Features**:
- **Lazy Allocation**: Created only when locks are needed
- **Type Separation**: Separate lists for each lock type
- **Atomic Installation**: Uses cmpxchg for race-free setup
- **RCU Integration**: Safe concurrent access and cleanup

### 2. Lock Allocation and Initialization

**Lock Allocation** - Lines 273-294:
```c
struct file_lock *locks_alloc_lock(void) {
    struct file_lock *fl = kmem_cache_zalloc(filelock_cache, GFP_KERNEL);
    
    if (fl)
        locks_init_lock_heads(&fl->c);
        
    return fl;
}

struct file_lease *locks_alloc_lease(void) {
    struct file_lease *fl = kmem_cache_zalloc(filelease_cache, GFP_KERNEL);
    
    if (fl)
        locks_init_lock_heads(&fl->c);
        
    return fl;
}
```

**Memory Management**:
- **SLAB Caches**: Efficient allocation from dedicated caches
- **Zero Initialization**: Ensures clean lock state
- **Head Initialization**: Sets up blocking and wait structures
- **Reference Counting**: Proper cleanup coordination

## POSIX Lock Implementation

### 1. POSIX Lock Processing

**Core POSIX Locking** - Lines 1146-1383:
```c
static int posix_lock_inode(struct inode *inode, struct file_lock *request,
                           struct file_lock *conflock) {
    struct file_lock *fl, *tmp;
    struct file_lock *new_fl = NULL;
    struct file_lock *new_fl2 = NULL;
    struct file_lock *left = NULL;
    struct file_lock *right = NULL;
    struct file_lock_context *ctx;
    int error;
    bool added = false;
    LIST_HEAD(dispose);
}
```

**POSIX Features**:
- **Byte Range Locking**: Supports precise range specifications
- **Lock Merging**: Adjacent/overlapping locks of same type merge
- **Lock Splitting**: Single locks can be split by conflicting ranges
- **Owner Detection**: Uses process files_struct for ownership
- **Deadlock Detection**: Sophisticated cycle detection algorithm

### 2. Deadlock Detection

**Deadlock Algorithm** - Lines 1024-1065:
```c
#define MAX_DEADLK_ITERATIONS 10

static bool posix_locks_deadlock(struct file_lock *caller_fl,
                                struct file_lock *block_fl) {
    struct file_lock_core *caller = &caller_fl->c;
    struct file_lock_core *blocker = &block_fl->c;
    int i = 0;
    
    if (caller->flc_flags & FL_OFDLCK)
        return false;
        
    while ((blocker = what_owner_is_waiting_for(blocker))) {
        if (i++ > MAX_DEADLK_ITERATIONS)
            return false;
        if (posix_same_owner(caller, blocker))
            return true;
    }
    return false;
}
```

**Deadlock Prevention**:
- **Chain Following**: Follows lock dependency chains
- **Cycle Detection**: Identifies circular wait conditions
- **Iteration Limiting**: Prevents infinite loops from broken clients
- **Owner Comparison**: Uses lock owner for cycle detection
- **OFD Exclusion**: OFD locks bypass deadlock detection

### 3. Lock Range Management

**Range Processing** - Lines 1238-1323:
```c
/* Process locks with this owner. */
list_for_each_entry_safe_from(fl, tmp, &ctx->flc_posix, c.flc_list) {
    if (!posix_same_owner(&request->c, &fl->c))
        break;
        
    /* Detect adjacent or overlapping regions (if same lock type) */
    if (request->c.flc_type == fl->c.flc_type) {
        if (fl->fl_end < request->fl_start - 1)
            continue;
        if (fl->fl_start - 1 > request->fl_end)
            break;
            
        /* Make one lock yielding from the lower start address */
        if (fl->fl_start > request->fl_start)
            fl->fl_start = request->fl_start;
        else
            request->fl_start = fl->fl_start;
    }
}
```

**Range Operations**:
- **Overlap Detection**: Identifies intersecting lock ranges
- **Merge Logic**: Combines adjacent or overlapping locks
- **Split Operations**: Divides locks when partially unlocked
- **Boundary Calculations**: Handles OFFSET_MAX edge cases

## FLOCK Implementation

### 1. BSD-Style Locking

**FLOCK Processing** - Lines 1074-1144:
```c
static int flock_lock_inode(struct inode *inode, struct file_lock *request) {
    struct file_lock *new_fl = NULL;
    struct file_lock *fl;
    struct file_lock_context *ctx;
    int error = 0;
    bool found = false;
    LIST_HEAD(dispose);
    
    ctx = locks_get_lock_context(inode, request->c.flc_type);
    if (!ctx) {
        if (request->c.flc_type != F_UNLCK)
            return -ENOMEM;
        return (request->c.flc_flags & FL_EXISTS) ? -ENOENT : 0;
    }
}
```

**FLOCK Characteristics**:
- **File-Level Locking**: Locks entire file, not byte ranges
- **Simple Semantics**: Only shared/exclusive/unlock operations
- **Per-File Conflicts**: Same file locks don't conflict with each other
- **Process Association**: Locked to specific file pointer
- **Non-Inheritable**: Not inherited across fork/exec

### 2. FLOCK Conflict Resolution

**Conflict Detection** - Lines 939-949:
```c
static bool flock_locks_conflict(struct file_lock_core *caller_flc,
                                struct file_lock_core *sys_flc) {
    /* FLOCK locks referring to the same filp do not conflict with
     * each other.
     */
    if (caller_flc->flc_file == sys_flc->flc_file)
        return false;
        
    return locks_conflict(caller_flc, sys_flc);
}
```

**Conflict Rules**:
- **Same File Exemption**: Multiple locks on same file descriptor allowed
- **Type-Based Conflicts**: Write locks conflict with all, read locks conflict with write
- **Simple Resolution**: First lock wins, others wait or fail

## Lease Implementation

### 1. File Leases

**Lease Management** - Lines 1542-1644:
```c
int __break_lease(struct inode *inode, unsigned int mode, unsigned int type) {
    int error = 0;
    struct file_lock_context *ctx;
    struct file_lease *new_fl, *fl, *tmp;
    unsigned long break_time;
    int want_write = (mode & O_ACCMODE) != O_RDONLY;
    LIST_HEAD(dispose);
    
    new_fl = lease_alloc(NULL, want_write ? F_WRLCK : F_RDLCK);
    if (IS_ERR(new_fl))
        return PTR_ERR(new_fl);
    new_fl->c.flc_flags = type;
}
```

**Lease Features**:
- **Opportunistic Locks**: Allow cached operations with conflict detection
- **Break Notifications**: SIGIO signals on lease conflicts
- **Timeout Management**: Configurable lease break timeouts
- **Delegation Support**: NFS delegation integration
- **Layout Support**: pNFS layout lease management

### 2. Lease Breaking Protocol

**Break Process** - Lines 1578-1594:
```c
list_for_each_entry_safe(fl, tmp, &ctx->flc_lease, c.flc_list) {
    if (!leases_conflict(&fl->c, &new_fl->c))
        continue;
    if (want_write) {
        if (fl->c.flc_flags & FL_UNLOCK_PENDING)
            continue;
        fl->c.flc_flags |= FL_UNLOCK_PENDING;
        fl->fl_break_time = break_time;
    } else {
        if (lease_breaking(fl))
            continue;
        fl->c.flc_flags |= FL_DOWNGRADE_PENDING;
        fl->fl_downgrade_time = break_time;
    }
    if (fl->fl_lmops->lm_break(fl))
        locks_delete_lock_ctx(&fl->c, &dispose);
}
```

**Break Protocol**:
- **Conflict Detection**: Identifies conflicting lease operations
- **Break Signaling**: Notifies lease holders via callback
- **Timeout Management**: Enforces maximum break time
- **Downgrade Support**: Read leases can be downgraded instead of broken
- **Async Processing**: Non-blocking break attempts with retry

## Block Management

### 1. Lock Blocking Infrastructure

**Block Insertion** - Lines 797-823:
```c
static void __locks_insert_block(struct file_lock_core *blocker,
                                struct file_lock_core *waiter,
                                bool conflict(struct file_lock_core *,
                                            struct file_lock_core *)) {
    struct file_lock_core *flc;
    
    BUG_ON(!list_empty(&waiter->flc_blocked_member));
new_blocker:
    list_for_each_entry(flc, &blocker->flc_blocked_requests, flc_blocked_member)
        if (conflict(flc, waiter)) {
            blocker = flc;
            goto new_blocker;
        }
    waiter->flc_blocker = blocker;
    list_add_tail(&waiter->flc_blocked_member,
                  &blocker->flc_blocked_requests);
}
```

**Blocking Features**:
- **Tree Structure**: Organizes waiters in conflict-based hierarchy
- **Conflict-Based Ordering**: Places waiters under blocking locks
- **Wake-up Chains**: Efficiently wakes appropriate waiters
- **Global Hash**: Enables efficient deadlock detection
- **Memory Ordering**: Uses proper barriers for lock-free access

### 2. Lock Wakeup Protocol

**Wakeup Processing** - Lines 700-724:
```c
static void __locks_wake_up_blocks(struct file_lock_core *blocker) {
    while (!list_empty(&blocker->flc_blocked_requests)) {
        struct file_lock_core *waiter;
        struct file_lock *fl;
        
        waiter = list_first_entry(&blocker->flc_blocked_requests,
                                 struct file_lock_core, flc_blocked_member);
                                 
        fl = file_lock(waiter);
        __locks_unlink_block(waiter);
        if ((waiter->flc_flags & (FL_POSIX | FL_FLOCK)) &&
            fl->fl_lmops && fl->fl_lmops->lm_notify)
            fl->fl_lmops->lm_notify(fl);
        else
            locks_wake_up(fl);
            
        smp_store_release(&waiter->flc_blocker, NULL);
    }
}
```

**Wakeup Protocol**:
- **Sequential Processing**: Wakes waiters in blocking order
- **Callback Support**: Calls lock manager notification if available
- **Memory Barriers**: Ensures proper ordering of blocker clearing
- **Recursive Wakeup**: Handles complex blocking hierarchies

## System Call Interface

### 1. flock() System Call

**FLOCK Implementation** - Lines 2135-2183:
```c
SYSCALL_DEFINE2(flock, unsigned int, fd, unsigned int, cmd) {
    int can_sleep, error, type;
    struct file_lock fl;
    
    if (cmd & LOCK_MAND) {
        pr_warn_once("%s(%d): Attempt to set a LOCK_MAND lock via flock(2). "
                    "This support has been removed and the request ignored.\n",
                    current->comm, current->pid);
        return 0;
    }
    
    type = flock_translate_cmd(cmd & ~LOCK_NB);
    if (type < 0)
        return type;
}
```

**FLOCK Features**:
- **Command Translation**: Converts user commands to kernel types
- **Non-blocking Support**: LOCK_NB flag support
- **Legacy Warnings**: Warns about deprecated LOCK_MAND usage
- **File Mode Checking**: Validates file access permissions
- **Security Integration**: LSM hook integration

### 2. fcntl() Lock Operations

**FCNTL Lock Processing** - Lines 2401-2475:
```c
int fcntl_setlk(unsigned int fd, struct file *filp, unsigned int cmd,
               struct flock *flock) {
    struct file_lock *file_lock = locks_alloc_lock();
    struct inode *inode = file_inode(filp);
    struct file *f;
    int error;
    
    if (file_lock == NULL)
        return -ENOLCK;
        
    error = flock_to_posix_lock(filp, file_lock, flock);
    if (error)
        goto out;
        
    error = check_fmode_for_setlk(file_lock);
    if (error)
        goto out;
}
```

**FCNTL Operations**:
- **F_SETLK/F_SETLKW**: Set or wait for POSIX locks
- **F_GETLK**: Query for conflicting locks
- **F_OFD_***: Open file description lock variants
- **Range Validation**: Checks lock range parameters
- **Race Detection**: Handles close/fcntl races

## Advanced Features

### 1. Open File Description Locks

**OFD Lock Support** - Lines 2424-2445:
```c
switch (cmd) {
case F_OFD_SETLK:
    error = -EINVAL;
    if (flock->l_pid != 0)
        goto out;
        
    cmd = F_SETLK;
    file_lock->c.flc_flags |= FL_OFDLCK;
    file_lock->c.flc_owner = filp;
    break;
case F_OFD_SETLKW:
    error = -EINVAL;
    if (flock->l_pid != 0)
        goto out;
        
    cmd = F_SETLKW;
    file_lock->c.flc_flags |= FL_OFDLCK;
    file_lock->c.flc_owner = filp;
    fallthrough;
}
```

**OFD Characteristics**:
- **File-Private Locks**: Associated with file description, not process
- **No Inheritance**: Not inherited across fork()
- **Deadlock Immune**: Bypass deadlock detection
- **Independent Ownership**: Separate from process-based ownership

### 2. Lock Translation and Validation

**Lock Format Conversion** - Lines 491-536:
```c
static int flock64_to_posix_lock(struct file *filp, struct file_lock *fl,
                                struct flock64 *l) {
    switch (l->l_whence) {
    case SEEK_SET:
        fl->fl_start = 0;
        break;
    case SEEK_CUR:
        fl->fl_start = filp->f_pos;
        break;
    case SEEK_END:
        fl->fl_start = i_size_read(file_inode(filp));
        break;
    default:
        return -EINVAL;
    }
    
    if (l->l_start > OFFSET_MAX - fl->fl_start)
        return -EOVERFLOW;
    fl->fl_start += l->l_start;
}
```

**Validation Features**:
- **Whence Processing**: Handles SEEK_SET/CUR/END correctly
- **Overflow Detection**: Prevents integer overflow attacks
- **Range Normalization**: Converts to absolute offsets
- **Negative Length Support**: POSIX-2001 negative length handling

## Memory Management and Cleanup

### 1. Lock Cleanup

**File Close Cleanup** - Lines 2685-2707:
```c
void locks_remove_file(struct file *filp) {
    struct file_lock_context *ctx;
    
    ctx = locks_inode_context(file_inode(filp));
    if (!ctx)
        return;
        
    /* remove any OFD locks */
    locks_remove_posix(filp, filp);
    
    /* remove flock locks */
    locks_remove_flock(filp, ctx);
    
    /* remove any leases */
    locks_remove_lease(filp, ctx);
    
    spin_lock(&ctx->flc_lock);
    locks_check_ctx_file_list(filp, &ctx->flc_posix, "POSIX");
    locks_check_ctx_file_list(filp, &ctx->flc_flock, "FLOCK");
    locks_check_ctx_file_list(filp, &ctx->flc_lease, "LEASE");
    spin_unlock(&ctx->flc_lock);
}
```

**Cleanup Protocol**:
- **Complete Removal**: Removes all lock types on file close
- **OFD Lock Cleanup**: Special handling for OFD locks
- **Leak Detection**: Debug checks for leaked locks
- **Order Independence**: Safe cleanup regardless of lock types

### 2. Memory Cache Management

**Cache Initialization** - Lines 2976-2998:
```c
static int __init filelock_init(void) {
    int i;
    
    flctx_cache = kmem_cache_create("file_lock_ctx",
                    sizeof(struct file_lock_context), 0, SLAB_PANIC, NULL);
                    
    filelock_cache = kmem_cache_create("file_lock_cache",
                    sizeof(struct file_lock), 0, SLAB_PANIC, NULL);
                    
    filelease_cache = kmem_cache_create("file_lease_cache",
                    sizeof(struct file_lease), 0, SLAB_PANIC, NULL);
                    
    for_each_possible_cpu(i) {
        struct file_lock_list_struct *fll = per_cpu_ptr(&file_lock_list, i);
        
        spin_lock_init(&fll->lock);
        INIT_HLIST_HEAD(&fll->hlist);
    }
    
    lease_notifier_chain_init();
    return 0;
}
```

**Memory Features**:
- **Dedicated Caches**: Separate SLAB caches for each structure type
- **Per-CPU Initialization**: Sets up per-CPU lock lists
- **Notifier Chains**: Initializes lease notification infrastructure
- **Early Initialization**: Core init call for early availability

## Security and Access Control

### 1. Permission Checking

**Lease Permission Validation** - Lines 2015-2028:
```c
int vfs_setlease(struct file *filp, int arg, struct file_lease **lease, void **priv) {
    struct inode *inode = file_inode(filp);
    vfsuid_t vfsuid = i_uid_into_vfsuid(file_mnt_idmap(filp), inode);
    int error;
    
    if ((!vfsuid_eq_kuid(vfsuid, current_fsuid())) && !capable(CAP_LEASE))
        return -EACCES;
    if (!S_ISREG(inode->i_mode))
        return -EINVAL;
    error = security_file_lock(filp, arg);
    if (error)
        return error;
    return kernel_setlease(filp, arg, lease, priv);
}
```

**Security Features**:
- **UID Validation**: Checks file owner or CAP_LEASE capability
- **File Type Validation**: Only regular files support leases
- **LSM Integration**: Security module hook for lock operations
- **Mount Namespace Support**: Proper UID mapping across namespaces

### 2. Race Condition Prevention

**Close/Lock Race Handling** - Lines 2449-2470:
```c
if (!error && file_lock->c.flc_type != F_UNLCK &&
    !(file_lock->c.flc_flags & FL_OFDLCK)) {
    struct files_struct *files = current->files;
    /*
     * We need that spin_lock here - it prevents reordering between
     * update of i_flctx->flc_posix and check for it done in
     * close(). rcu_read_lock() wouldn't do.
     */
    spin_lock(&files->file_lock);
    f = files_lookup_fd_locked(files, fd);
    spin_unlock(&files->file_lock);
    if (f != filp) {
        locks_remove_posix(filp, files);
        error = -EBADF;
    }
}
```

**Race Prevention**:
- **Memory Ordering**: Spin locks prevent dangerous reordering
- **FD Validation**: Ensures file descriptor hasn't been closed/reused
- **Lock Cleanup**: Removes locks if file descriptor changed
- **OFD Exception**: OFD locks immune to close/fcntl races

## Observability and Debugging

### 1. /proc/locks Interface

**Lock Display** - Lines 2852-2895:
```c
static int locks_show(struct seq_file *f, void *v) {
    struct locks_iterator *iter = f->private;
    struct file_lock_core *cur, *tmp;
    struct pid_namespace *proc_pidns = proc_pid_ns(file_inode(f->file)->i_sb);
    int level = 0;
    
    cur = hlist_entry(v, struct file_lock_core, flc_link);
    
    if (locks_translate_pid(cur, proc_pidns) == 0)
        return 0;
}
```

**Display Features**:
- **Hierarchical Display**: Shows lock blocking relationships
- **PID Translation**: Translates PIDs to current namespace
- **Lock Type Identification**: Distinguishes POSIX, FLOCK, LEASE
- **Status Information**: Shows lock state and ranges
- **Tree Traversal**: Binary tree traversal of blocking relationships

### 2. Lock Leak Detection

**Debug Infrastructure** - Lines 208-250:
```c
static void locks_check_ctx_lists(struct inode *inode) {
    struct file_lock_context *ctx = inode->i_flctx;
    
    if (unlikely(!list_empty(&ctx->flc_flock) ||
                !list_empty(&ctx->flc_posix) ||
                !list_empty(&ctx->flc_lease))) {
        pr_warn("Leaked locks on dev=0x%x:0x%x ino=0x%lx:\n",
               MAJOR(inode->i_sb->s_dev), MINOR(inode->i_sb->s_dev),
               inode->i_ino);
        locks_dump_ctx_list(&ctx->flc_flock, "FLOCK");
        locks_dump_ctx_list(&ctx->flc_posix, "POSIX");
        locks_dump_ctx_list(&ctx->flc_lease, "LEASE");
    }
}
```

**Debug Support**:
- **Leak Detection**: Identifies unreleased locks during cleanup
- **Lock Dumping**: Provides detailed information about leaked locks
- **Device Information**: Shows device and inode numbers
- **Type Classification**: Separates different lock types in output

## Performance Optimizations

### 1. Per-CPU Data Structures

**CPU-Local Lists**: 
- **Reduced Contention**: Per-CPU lock lists for /proc display
- **NUMA Awareness**: Respects CPU topology
- **Scalable Access**: O(1) access to local data
- **Memory Locality**: Keeps related data CPU-local

### 2. Lock-Free Operations

**RCU Protection**:
- **Reader Scalability**: Multiple readers without locks
- **Grace Periods**: Safe memory reclamation
- **Atomic Updates**: Lock-free state transitions
- **Memory Barriers**: Proper ordering guarantees

### 3. Efficient Algorithms

**Tree-Based Organization**:
- **Conflict Hierarchy**: Organizes waiters by conflict relationships
- **Fast Wakeup**: Wakes only relevant waiters
- **Deadlock Detection**: Efficient cycle detection
- **Range Merging**: Optimizes overlapping lock ranges

## Integration Points

### 1. Virtual File System (VFS)

**VFS Coordination**:
- **File Operations**: Integrates with VFS file operations
- **Inode Association**: Links locks to filesystem inodes
- **Security Integration**: LSM hook integration
- **Mount Namespace Support**: Proper namespace handling

### 2. Process Management

**Process Integration**:
- **Task Association**: Links locks to processes and threads
- **Inheritance Semantics**: Manages lock inheritance across fork/exec
- **Signal Integration**: SIGIO delivery for lease breaks
- **Files Structure**: Coordinates with process file tables

### 3. Network File Systems

**NFS Integration**:
- **Delegation Support**: Implements NFS delegation semantics
- **Remote Locking**: Supports distributed lock protocols
- **Callback Mechanisms**: Handles remote lock notifications
- **State Recovery**: Manages lock state across network partitions

The file locking system represents one of the most complex subsystems in the Linux kernel, providing sophisticated locking semantics while maintaining compatibility with multiple standards and use cases. Its careful handling of deadlock detection, race conditions, and performance optimization makes it a critical foundation for reliable concurrent file access across the entire system.
