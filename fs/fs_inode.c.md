# fs/inode.c - Linux Inode Management and Virtual File System Core

## Overview

This file implements the core inode management system for the Linux Virtual File System (VFS), originally developed by Linus Torvalds in 1997 with significant enhancements by Andrea Arcangeli in 1999 for dynamic inode allocation. The inode subsystem provides the fundamental abstraction for file system objects, managing file metadata, implementing reference counting, and providing the interface between generic VFS operations and specific file system implementations.

## Historical Development

### Key Contributors and Milestones
- **Linus Torvalds (1997)**: Original inode management implementation
- **Andrea Arcangeli - SUSE (1999)**: Dynamic inode allocation system
- **Community Contributors**: Ongoing scalability and performance improvements

### Evolution Timeline
- **1997**: Basic inode management and VFS abstraction
- **1999**: Dynamic inode allocation and improved memory management
- **2000s**: Scalability improvements and RCU integration
- **2010s**: Per-CPU statistics, LRU management, and writeback integration
- **2020s**: Modern security features and container support

### Design Philosophy
The inode system is designed around the principle of providing a uniform interface to diverse file system implementations while maintaining optimal performance through caching, reference counting, and efficient lookup mechanisms.

## Core Concepts

### Inode Architecture

#### VFS Inode Abstraction
```
File System Objects → Generic Inodes → VFS Operations → User Space
        ↓                  ↓              ↓             ↓
[FS-Specific Data]  [Common Metadata]  [System Calls] [Applications]
```

#### Inode Lifecycle States
```
Allocation → Initialization → Active Use → Eviction → Destruction
     ↓            ↓             ↓          ↓           ↓
[new_inode]  [inode_init]  [Hash Table] [LRU List] [destroy_inode]
```

#### Inode Types and Classification
- **Regular Files**: Standard data files with content
- **Directories**: File system navigation structures
- **Symbolic Links**: File system redirection objects
- **Device Files**: Hardware device interfaces (block/character)
- **Special Files**: FIFOs, sockets, and other special objects

## Key Data Structures

### Inode State Management
```c
/* Inode state flags */
#define I_DIRTY_SYNC     (1 << 0)   /* Inode metadata dirty */
#define I_DIRTY_DATASYNC (1 << 1)   /* Data dirty */
#define I_DIRTY_PAGES    (1 << 2)   /* Data pages dirty */
#define I_NEW            (1 << 3)   /* Inode being created */
#define I_WILL_FREE      (1 << 4)   /* Inode will be freed */
#define I_FREEING        (1 << 5)   /* Inode being freed */
#define I_CLEAR          (1 << 6)   /* Inode cleared */
#define I_SYNC           (1 << 7)   /* Sync in progress */
#define I_CREATING       (1 << 8)   /* Inode creation in progress */
#define I_LINKABLE       (1 << 9)   /* Inode can be linked */

#define I_DIRTY_INODE (I_DIRTY_SYNC | I_DIRTY_DATASYNC)
#define I_DIRTY (I_DIRTY_INODE | I_DIRTY_PAGES)
#define I_DIRTY_ALL (I_DIRTY | I_DIRTY_TIME)
```

### Inode Locking Hierarchy
```c
/*
 * Lock ordering:
 *
 * inode->i_sb->s_inode_list_lock
 *   inode->i_lock
 *     Inode LRU list locks
 *
 * bdi->wb.list_lock
 *   inode->i_lock
 *
 * inode_hash_lock
 *   inode->i_sb->s_inode_list_lock
 *   inode->i_lock
 */
```

### Inode Hash Table Management
```c
static unsigned int i_hash_mask __ro_after_init;
static unsigned int i_hash_shift __ro_after_init;
static struct hlist_head *inode_hashtable __ro_after_init;
static __cacheline_aligned_in_smp DEFINE_SPINLOCK(inode_hash_lock);
```

**Hash Table Features**:
- **Scalable Lookup**: Efficient inode lookup by (superblock, inode number)
- **RCU Protection**: Lock-free read access for improved scalability
- **Cache-line Aligned**: Optimized for SMP performance
- **Dynamic Sizing**: Hash table size scales with system memory

### Per-CPU Statistics
```c
static DEFINE_PER_CPU(unsigned long, nr_inodes);
static DEFINE_PER_CPU(unsigned long, nr_unused);
```

**Statistical Tracking**:
- **Total Inodes**: System-wide inode count
- **Unused Inodes**: Inodes eligible for reclaim
- **Per-CPU Counters**: Avoid cache line bouncing
- **Overflow Protection**: Proper handling of counter overflow

## Core Functions

### Inode Allocation and Initialization

#### `alloc_inode()` - Core Inode Allocation
```c
struct inode *alloc_inode(struct super_block *sb)
```

**Purpose**: Allocate a new inode for the specified superblock

**Allocation Process**:
1. **File System Callback**: Use filesystem-specific allocator if available
2. **Generic Allocation**: Fall back to generic inode cache allocation
3. **Initialization**: Initialize inode structure and metadata
4. **Error Handling**: Clean up on initialization failure

**Key Features**:
```c
if (ops->alloc_inode)
    inode = ops->alloc_inode(sb);        /* FS-specific allocation */
else
    inode = alloc_inode_sb(sb, inode_cachep, GFP_KERNEL); /* Generic allocation */

if (unlikely(inode_init_always(sb, inode))) {
    /* Handle initialization failure */
    if (ops->destroy_inode) {
        ops->destroy_inode(inode);
        if (!ops->free_inode)
            return NULL;
    }
    inode->free_inode = ops->free_inode;
    i_callback(&inode->i_rcu);
    return NULL;
}
```

#### `new_inode()` - Public Inode Creation Interface
```c
struct inode *new_inode(struct super_block *sb)
```

**Public Interface Features**:
- **Superblock Integration**: Add inode to superblock inode list
- **Memory Policy**: Default GFP_HIGHUSER_MOVABLE for page cache
- **Statistics Update**: Update per-CPU inode counters
- **Reference Management**: Establish initial reference count

#### `inode_init_always()` - Inode Structure Initialization
**Purpose**: Initialize all common inode fields and structures

**Initialization Components**:
1. **Basic Metadata**: UID, GID, permissions, timestamps
2. **Locking Structures**: Initialize locks and synchronization primitives
3. **Address Space**: Set up address space for file data
4. **List Initialization**: Initialize various list heads
5. **Security Context**: Initialize security and audit contexts

### Inode Destruction and Cleanup

#### `__destroy_inode()` - Core Inode Destruction
```c
void __destroy_inode(struct inode *inode)
```

**Destruction Process**:
1. **Buffer Validation**: Ensure no buffers remain attached
2. **Writeback Detachment**: Detach from writeback system
3. **Security Cleanup**: Clean up security contexts
4. **Notification**: Notify file system notification systems
5. **Lock Cleanup**: Clean up file locking contexts
6. **Reference Accounting**: Update remove count for statistics

**Security and Resource Cleanup**:
```c
BUG_ON(inode_has_buffers(inode));       /* Validate clean state */
inode_detach_wb(inode);                 /* Writeback detachment */
security_inode_free(inode);             /* Security cleanup */
fsnotify_inode_delete(inode);           /* Notification cleanup */
locks_free_lock_context(inode);         /* File lock cleanup */

#ifdef CONFIG_FS_POSIX_ACL
if (inode->i_acl && !is_uncached_acl(inode->i_acl))
    posix_acl_release(inode->i_acl);    /* ACL cleanup */
if (inode->i_default_acl && !is_uncached_acl(inode->i_default_acl))
    posix_acl_release(inode->i_default_acl);
#endif
```

#### `destroy_inode()` - Public Destruction Interface
```c
static void destroy_inode(struct inode *inode)
```

**Public Interface Features**:
- **File System Callback**: Use filesystem-specific destructor
- **RCU Protection**: Use RCU for safe delayed destruction
- **LRU Validation**: Ensure inode is not on LRU lists
- **Memory Management**: Coordinate with memory management

### Inode Hash Table Operations

#### `find_inode()` - Generic Inode Lookup
```c
static struct inode *find_inode(struct super_block *sb,
                               struct hlist_head *head,
                               int (*test)(struct inode *, void *),
                               void *data, bool is_inode_hash_locked)
```

**Lookup Process**:
1. **RCU Read Section**: Enter RCU read-side critical section
2. **Hash Chain Traversal**: Search hash chain for matching inode
3. **Custom Test Function**: Use filesystem-provided test function
4. **State Validation**: Handle inodes in transitional states
5. **Reference Acquisition**: Safely acquire reference to found inode

**Concurrency Handling**:
```c
rcu_read_lock();
repeat:
hlist_for_each_entry_rcu(inode, head, i_hash) {
    if (inode->i_sb != sb)
        continue;
    if (!test(inode, data))
        continue;
    spin_lock(&inode->i_lock);
    if (inode->i_state & (I_FREEING|I_WILL_FREE)) {
        __wait_on_freeing_inode(inode, is_inode_hash_locked);
        goto repeat;
    }
    if (unlikely(inode->i_state & I_CREATING)) {
        spin_unlock(&inode->i_lock);
        rcu_read_unlock();
        return ERR_PTR(-ESTALE);
    }
    __iget(inode);
    spin_unlock(&inode->i_lock);
    rcu_read_unlock();
    return inode;
}
```

#### `find_inode_fast()` - Optimized Inode Number Lookup
```c
static struct inode *find_inode_fast(struct super_block *sb,
                                    struct hlist_head *head, unsigned long ino,
                                    bool is_inode_hash_locked)
```

**Fast Path Optimizations**:
- **Simple Comparison**: Direct inode number comparison
- **No Custom Test**: Avoid function call overhead
- **Common Case**: Optimized for typical file system usage
- **Lock-free Read**: RCU-protected hash table traversal

### Inode Reference Management

#### `__iget()` - Internal Reference Increment
**Purpose**: Safely increment inode reference count

**Reference Counting Features**:
- **Atomic Operations**: Thread-safe reference counting
- **State Validation**: Ensure inode is in valid state
- **Statistics Integration**: Update usage statistics
- **Memory Barriers**: Proper memory ordering

#### `iput()` - Reference Decrement and Cleanup
**Purpose**: Decrement reference count and clean up if necessary

**Cleanup Process**:
1. **Reference Decrement**: Atomically decrement reference count
2. **Zero Reference Handling**: Handle transition to zero references
3. **LRU Management**: Move to LRU list or destroy immediately
4. **Final Cleanup**: Perform final cleanup if no references remain

### Inode Number Management

#### `get_next_ino()` - Inode Number Generation
```c
unsigned int get_next_ino(void)
```

**Number Generation Strategy**:
- **Per-CPU Ranges**: Each CPU has its own range of inode numbers
- **Batch Allocation**: Reduce contention through batch allocation
- **Overflow Handling**: Graceful handling of number space exhaustion
- **32-bit Compatibility**: Consider 32-bit stat() compatibility

**Per-CPU Optimization**:
```c
#define LAST_INO_BATCH 1024
static DEFINE_PER_CPU(unsigned int, last_ino);

unsigned int get_next_ino(void)
{
    unsigned int *p = &get_cpu_var(last_ino);
    unsigned int res = *p;

    if (unlikely((res & (LAST_INO_BATCH-1)) == 0)) {
        static atomic_t shared_last_ino;
        int next = atomic_add_return(LAST_INO_BATCH, &shared_last_ino);

        res = next - LAST_INO_BATCH;
    }
    res++;
    *p = res;
    put_cpu_var(last_ino);
    return res;
}
```

### Link Count Management

#### `drop_nlink()` - Decrement Link Count
```c
void drop_nlink(struct inode *inode)
```

**Link Management Features**:
- **Validation**: Ensure link count doesn't underflow
- **Remove Tracking**: Track files pending removal
- **Atomic Updates**: Thread-safe link count modification
- **Statistics Integration**: Update removal statistics

#### `clear_nlink()` - Zero Link Count
```c
void clear_nlink(struct inode *inode)
```

**Zero Link Features**:
- **Direct Zeroing**: Direct link count zeroing
- **Removal Tracking**: Mark for pending removal
- **Cleanup Coordination**: Coordinate with cleanup subsystems
- **File System Integration**: Support file system-specific behavior

### Inode State Management

#### `unlock_new_inode()` - Complete Inode Initialization
```c
void unlock_new_inode(struct inode *inode)
```

**Initialization Completion**:
1. **Lockdep Annotation**: Set up lockdep classes for directory inodes
2. **State Clearing**: Clear I_NEW and I_CREATING states
3. **Memory Barriers**: Ensure proper memory ordering
4. **Waiter Notification**: Wake up any threads waiting for initialization

**State Transition Handling**:
```c
lockdep_annotate_inode_mutex_key(inode);
spin_lock(&inode->i_lock);
WARN_ON(!(inode->i_state & I_NEW));
inode->i_state &= ~I_NEW & ~I_CREATING;
smp_mb();                               /* Memory barrier */
inode_wake_up_bit(inode, __I_NEW);     /* Wake waiters */
spin_unlock(&inode->i_lock);
```

#### `discard_new_inode()` - Discard Failed Inode
```c
void discard_new_inode(struct inode *inode)
```

**Discard Process**:
- **State Cleanup**: Clean up I_NEW state
- **Waiter Notification**: Notify waiting threads
- **Reference Drop**: Drop the initial reference
- **Resource Cleanup**: Ensure proper resource cleanup

## Advanced Features

### Inode LRU Management

#### LRU List Integration
- **Unused Inode Tracking**: Track inodes eligible for reclaim
- **Age-based Eviction**: Evict oldest unused inodes first
- **Memory Pressure Response**: Respond to memory pressure
- **Writeback Coordination**: Coordinate with writeback before eviction

#### `prune_icache_sb()` - Superblock Inode Pruning
**Purpose**: Prune unused inodes from specific superblock

**Pruning Features**:
- **Selective Pruning**: Prune only from specific superblock
- **Writeback Handling**: Handle dirty inodes appropriately
- **Reference Validation**: Ensure inodes are truly unused
- **Statistics Update**: Update pruning statistics

### Writeback Integration

#### Dirty Inode Management
- **Dirty State Tracking**: Track various dirty states
- **Writeback Coordination**: Coordinate with writeback subsystem
- **Congestion Handling**: Handle storage device congestion
- **Error Propagation**: Propagate writeback errors

#### `mark_inode_dirty()` - Mark Inode for Writeback
**Purpose**: Mark inode as needing writeback

**Dirty Marking Process**:
1. **State Classification**: Determine type of dirtiness
2. **List Management**: Add to appropriate writeback lists
3. **Congestion Check**: Check for writeback congestion
4. **Scheduling**: Schedule writeback if appropriate

### Security and Access Control

#### POSIX ACL Support
```c
#ifdef CONFIG_FS_POSIX_ACL
if (inode->i_acl && !is_uncached_acl(inode->i_acl))
    posix_acl_release(inode->i_acl);
if (inode->i_default_acl && !is_uncached_acl(inode->i_default_acl))
    posix_acl_release(inode->i_default_acl);
#endif
```

**ACL Features**:
- **Access Control Lists**: Support for extended permissions
- **Inheritance**: Default ACL inheritance for directories
- **Caching**: Efficient ACL caching mechanisms
- **Cleanup**: Proper ACL cleanup on inode destruction

#### Security Framework Integration
- **LSM Integration**: Linux Security Module support
- **Capability Checking**: File capability support
- **Audit Integration**: File access audit support
- **Namespace Support**: User namespace integration

### Performance Optimizations

#### Hash Table Optimizations
- **Cache-line Alignment**: Optimize for CPU cache performance
- **RCU Protection**: Lock-free read access
- **Dynamic Sizing**: Scale with system size
- **Load Balancing**: Distribute load across hash buckets

#### Memory Management Optimizations
- **Slab Allocation**: Efficient memory allocation
- **Per-CPU Statistics**: Reduce cache line bouncing
- **RCU Synchronization**: Efficient synchronization
- **Memory Reclaim**: Coordinate with memory reclaim

### Debugging and Monitoring

#### Statistics Collection
```c
long get_nr_inodes(void)
{
    int i;
    long sum = 0;
    for_each_possible_cpu(i)
        sum += per_cpu(nr_inodes, i);
    return sum < 0 ? 0 : sum;
}
```

**Monitoring Features**:
- **System Statistics**: Total inode counts
- **Usage Statistics**: Active vs. unused inodes
- **Per-Superblock Statistics**: Filesystem-specific counts
- **Debug Information**: Detailed debugging support

#### Lock Debugging
- **Lockdep Integration**: Deadlock detection support
- **Lock Annotation**: Proper lock class annotation
- **State Validation**: Comprehensive state validation
- **Race Detection**: Detection of common race conditions

## Integration Points

### Virtual File System Integration
- **VFS Operations**: Generic file operations interface
- **Superblock Integration**: Tight integration with superblock management
- **Dentry Cache**: Coordination with directory entry cache
- **Page Cache**: Integration with file data page cache

### File System Integration
- **Generic Interface**: Common interface for all file systems
- **Custom Operations**: Support for filesystem-specific operations
- **Metadata Management**: Generic metadata handling
- **Error Handling**: Consistent error handling across file systems

### Memory Management Integration
- **Page Cache**: File data page management
- **Memory Reclaim**: Integration with memory reclaim
- **Writeback**: Dirty data writeback coordination
- **Memory Pressure**: Response to memory pressure

### Security Framework Integration
- **Access Control**: File access permission checking
- **Audit System**: File access auditing
- **Namespace Support**: Container and namespace integration
- **Capability System**: File capability support

This comprehensive inode management implementation provides the foundation for file system operations in Linux, enabling efficient, scalable, and secure file access while maintaining consistency across diverse file system implementations and supporting modern features like security frameworks, containers, and high-performance storage devices.