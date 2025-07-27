# fs/dcache.c - Linux Directory Entry Cache Implementation

## Overview

This file implements the directory entry cache (dcache) for the Linux Virtual File System, originally designed and implemented by Thomas Schoebel-Theuer in 1997 with significant modifications by Linus Torvalds. The dcache provides a critical caching layer for directory entries, enabling fast pathname resolution and maintaining the relationship between file names and their corresponding inodes. It serves as the master of the inode cache, ensuring that inodes remain accessible as long as directory entries reference them.

## Historical Development

### Key Contributors and Milestones
- **Thomas Schoebel-Theuer (1997)**: Complete reimplementation of dcache
- **Linus Torvalds**: Heavy modifications and optimizations
- **Community Contributors**: Ongoing scalability and RCU improvements

### Evolution Timeline
- **1997**: Complete dcache reimplementation with modern architecture
- **2000s**: RCU integration and lock-free path walking
- **2010s**: Scalability improvements and per-CPU statistics
- **2020s**: Advanced security features and container support

### Design Philosophy
The dcache is designed around the principle that directory entries are the primary interface to the file system, with inodes serving as the underlying storage for file metadata. This master-slave relationship ensures efficient memory management and consistent file system state.

## Core Concepts

### Directory Cache Architecture

#### Cache Hierarchy
```
Path Resolution → Dcache Lookup → Inode Access → File Operations
       ↓              ↓             ↓              ↓
[Component Names] [Hash Table]  [Metadata]   [Data Access]
```

#### Dcache Lifecycle
```
Allocation → Hashing → Active Use → LRU Management → Reclaim → Destruction
     ↓          ↓          ↓           ↓             ↓          ↓
[d_alloc]  [d_hash]   [Lookup]   [LRU Lists]   [Shrinking] [RCU Free]
```

#### Entry Types and States
- **Positive Entries**: Directory entries with associated inodes
- **Negative Entries**: Cached lookup failures (file doesn't exist)
- **Anonymous Entries**: Disconnected entries for special files
- **Cursor Entries**: Special entries for directory iteration

## Key Data Structures

### Dentry Structure Core Components
```c
struct dentry {
    /* Reference counting and locking */
    struct lockref d_lockref;          /* Reference count + spinlock */
    seqcount_spinlock_t d_seq;         /* Sequence lock for RCU */
    
    /* Hash table linkage */
    struct hlist_bl_node d_hash;       /* Hash chain linkage */
    
    /* Name and identification */
    struct qstr d_name;                /* Name structure */
    char d_shortname[DNAME_INLINE_LEN]; /* Inline name storage */
    
    /* Hierarchy relationships */
    struct dentry *d_parent;           /* Parent directory */
    struct hlist_head d_children;      /* Child dentries */
    struct hlist_node d_sib;           /* Sibling linkage */
    
    /* Inode association */
    struct inode *d_inode;             /* Associated inode */
    struct hlist_node d_u.d_alias;     /* Inode alias list */
    
    /* File system integration */
    struct super_block *d_sb;          /* Superblock */
    const struct dentry_operations *d_op; /* Operations */
    void *d_fsdata;                    /* FS-specific data */
    
    /* LRU and state management */
    struct list_head d_lru;            /* LRU list linkage */
    unsigned int d_flags;              /* State flags */
};
```

### Name Structure (qstr)
```c
struct qstr {
    union {
        struct {
            HASH_LEN_DECLARE;          /* Hash and length */
        };
        u64 hash_len;                  /* Combined hash/length */
    };
    const unsigned char *name;         /* Name string */
};
```

### Dentry State Flags
```c
#define DCACHE_OP_HASH          0x00000001  /* Custom hash function */
#define DCACHE_OP_COMPARE       0x00000002  /* Custom compare function */
#define DCACHE_OP_REVALIDATE    0x00000004  /* Revalidation required */
#define DCACHE_OP_DELETE        0x00000008  /* Custom delete function */
#define DCACHE_OP_PRUNE         0x00000010  /* Custom prune function */

#define DCACHE_DISCONNECTED     0x00000020  /* Disconnected from tree */
#define DCACHE_REFERENCED       0x00000040  /* Recently referenced */
#define DCACHE_DONTCACHE        0x00000080  /* Don't cache entry */

#define DCACHE_CANT_MOUNT       0x00000100  /* Cannot be mount point */
#define DCACHE_GENOCIDE         0x00000200  /* Being killed recursively */
#define DCACHE_SHRINK_LIST      0x00000400  /* On shrink list */
#define DCACHE_OP_WEAK_REVALIDATE 0x00000800 /* Weak revalidation */

#define DCACHE_NFSFS_RENAMED    0x00001000  /* NFS rename optimization */
#define DCACHE_COOKIE           0x00002000  /* FS-Cache cookie */
#define DCACHE_FSNOTIFY_PARENT_WATCHED 0x00004000 /* Parent watched */
#define DCACHE_DENTRY_KILLED    0x00008000  /* Dentry killed */

#define DCACHE_MOUNTED          0x00010000  /* Mount point */
#define DCACHE_NEED_AUTOMOUNT   0x00020000  /* Automount required */
#define DCACHE_MANAGE_TRANSIT   0x00040000  /* Transit management */
#define DCACHE_MANAGED_DENTRY   (DCACHE_MOUNTED|DCACHE_NEED_AUTOMOUNT|DCACHE_MANAGE_TRANSIT)

#define DCACHE_LRU_LIST         0x00080000  /* On LRU list */
#define DCACHE_ENTRY_TYPE       0x00700000  /* Entry type mask */
#define DCACHE_MISS_TYPE        0x00000000  /* Negative entry */
#define DCACHE_WHITEOUT_TYPE    0x00100000  /* Whiteout entry */
#define DCACHE_DIRECTORY_TYPE   0x00200000  /* Directory entry */
#define DCACHE_AUTODIR_TYPE     0x00300000  /* Auto-created directory */
#define DCACHE_REGULAR_TYPE     0x00400000  /* Regular file entry */
#define DCACHE_SPECIAL_TYPE     0x00500000  /* Special file entry */
#define DCACHE_SYMLINK_TYPE     0x00600000  /* Symbolic link entry */
```

### Locking Architecture
```c
/*
 * Locking hierarchy:
 * dcache_hash_bucket lock protects:
 *   - the dcache hash table
 * s_roots bl list spinlock protects:
 *   - the s_roots list
 * dentry->d_sb->s_dentry_lru_lock protects:
 *   - the dcache lru lists and counters
 * d_lock protects:
 *   - d_flags, d_name, d_lru, d_count
 *   - d_unhashed(), d_parent and d_children
 *   - children's d_sib and d_parent
 *   - d_u.d_alias, d_inode
 *
 * Ordering:
 * dentry->d_inode->i_lock
 *   dentry->d_lock
 *     dentry->d_sb->s_dentry_lru_lock
 *     dcache_hash_bucket lock
 *     s_roots lock
 */
```

## Core Functions

### Dentry Allocation and Initialization

#### `__d_alloc()` - Core Dentry Allocation
```c
static struct dentry *__d_alloc(struct super_block *sb, const struct qstr *name)
```

**Purpose**: Allocate and initialize a new directory entry

**Allocation Process**:
1. **Memory Allocation**: Allocate from dentry cache with LRU integration
2. **Name Management**: Handle inline vs. external name storage
3. **Structure Initialization**: Initialize all dentry fields
4. **Operations Setup**: Configure filesystem-specific operations
5. **Statistics Update**: Update per-CPU dentry counters

**Name Storage Optimization**:
```c
/* Inline name storage for short names */
dentry->d_shortname.string[DNAME_INLINE_LEN-1] = 0;
if (name->len > DNAME_INLINE_LEN-1) {
    /* External storage for long names */
    size_t size = offsetof(struct external_name, name[1]);
    struct external_name *p = kmalloc(size + name->len,
                                     GFP_KERNEL_ACCOUNT | __GFP_RECLAIMABLE);
    atomic_set(&p->count, 1);
    dname = p->name;
} else {
    dname = dentry->d_shortname.string;
}
```

**Reference and Lock Initialization**:
```c
lockref_init(&dentry->d_lockref);           /* Combined lock+refcount */
seqcount_spinlock_init(&dentry->d_seq, &dentry->d_lock); /* RCU seqlock */
dentry->d_inode = NULL;                     /* No inode initially */
dentry->d_parent = dentry;                  /* Self-parent initially */
```

#### `d_alloc()` - Public Dentry Allocation Interface
```c
struct dentry *d_alloc(struct dentry *parent, const struct qstr *name)
```

**Public Interface Features**:
- **Parent Relationship**: Establish parent-child relationship
- **Hierarchy Integration**: Add to parent's children list
- **Reference Management**: Proper reference counting
- **Locking Coordination**: Acquire parent lock for hierarchy updates

#### Specialized Allocation Functions
```c
struct dentry *d_alloc_anon(struct super_block *sb);     /* Anonymous entries */
struct dentry *d_alloc_cursor(struct dentry *parent);   /* Directory cursors */
struct dentry *d_alloc_pseudo(struct super_block *sb, const struct qstr *name); /* Pseudo entries */
```

### Dentry Lookup Operations

#### `__d_lookup_rcu()` - RCU-Protected Lookup
```c
struct dentry *__d_lookup_rcu(const struct dentry *parent,
                             const struct qstr *name,
                             unsigned *seqp)
```

**Purpose**: Perform lock-free dentry lookup using RCU protection

**RCU Lookup Process**:
1. **Hash Calculation**: Compute hash for efficient lookup
2. **RCU Traversal**: Traverse hash chain under RCU protection
3. **Sequence Validation**: Validate dentry consistency with seqlock
4. **Name Comparison**: Compare names with proper memory ordering
5. **Result Validation**: Return sequence number for caller validation

**Concurrency Safety**:
```c
/* RCU-protected hash chain traversal */
hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {
    unsigned seq;
    
    /* Raw seqcount for performance - caller must validate */
    seq = raw_seqcount_begin(&dentry->d_seq);
    if (dentry->d_parent != parent)
        continue;
    if (d_unhashed(dentry))
        continue;
    if (dentry->d_name.hash_len != hashlen)
        continue;
    if (dentry_cmp(dentry, str, hashlen_len(hashlen)) != 0)
        continue;
    *seqp = seq;
    return dentry;
}
```

#### `d_lookup()` - Standard Dentry Lookup
```c
struct dentry *d_lookup(const struct dentry *parent, const struct qstr *name)
```

**Standard Lookup Features**:
- **Reference Acquisition**: Acquire reference to found dentry
- **Rename Protection**: Protection against concurrent renames
- **Comprehensive Search**: Search all children of parent
- **Reference Counting**: Proper reference management

#### Hash Table Management

#### `d_hash()` - Hash Function
```c
static inline struct hlist_bl_head *d_hash(unsigned long hashlen)
{
    return runtime_const_ptr(dentry_hashtable) +
           runtime_const_shift_right_32(hashlen, d_hash_shift);
}
```

**Hash Table Features**:
- **Runtime Constants**: Optimized hash calculation
- **Load Distribution**: Even distribution across buckets
- **Scalable Size**: Hash table size scales with system memory
- **Collision Handling**: Efficient collision resolution

### Dentry Reference Management

#### Reference Counting with Lockref
```c
struct lockref {
    union {
        struct {
            spinlock_t lock;
            int count;
        };
        aligned_u64 lock_count;
    };
};
```

**Lockref Benefits**:
- **Atomic Operations**: Combined lock and reference operations
- **Cache Efficiency**: Single cache line for lock and count
- **Scalability**: Reduced lock contention
- **Performance**: Optimized common case operations

#### `dget()` - Acquire Dentry Reference
**Purpose**: Safely acquire reference to dentry

#### `dput()` - Release Dentry Reference
**Purpose**: Release reference and potentially free dentry

### Dentry Hash Operations

#### `d_hash_and_lookup()` - Combined Hash and Lookup
**Purpose**: Efficiently combine hashing and lookup operations

#### `__d_drop()` - Remove from Hash Table
**Purpose**: Remove dentry from hash table (unhash)

#### `d_rehash()` - Add to Hash Table
**Purpose**: Add dentry to hash table

### LRU Management and Reclaim

#### LRU List Management
```c
static void d_lru_add(struct dentry *dentry)
static void d_lru_del(struct dentry *dentry)
static void d_lru_shrink_move(struct list_lru_one *list, struct dentry *dentry,
                             struct list_head *dispose)
```

**LRU Features**:
- **Age-based Reclaim**: Reclaim oldest unused dentries
- **Pressure Response**: Respond to memory pressure
- **Statistics Tracking**: Track LRU statistics
- **Selective Reclaim**: Per-superblock LRU management

#### Dentry Shrinker Integration
```c
static unsigned long shrink_dcache_count(struct shrinker *shrink,
                                        struct shrink_control *sc)
static unsigned long shrink_dcache_scan(struct shrinker *shrink,
                                       struct shrink_control *sc)
```

**Shrinker Features**:
- **Memory Pressure**: Respond to kernel memory pressure
- **Proportional Reclaim**: Reclaim proportional to pressure
- **NUMA Awareness**: NUMA-aware reclaim decisions
- **Statistics Integration**: Provide accurate counts

### Negative Dentry Management

#### Negative Dentry Caching
**Purpose**: Cache failed lookups to avoid repeated file system operations

**Benefits**:
- **Performance**: Avoid repeated negative lookups
- **Reduced I/O**: Minimize file system operations
- **Scalability**: Improve performance under heavy lookup load
- **Memory Efficiency**: Balanced caching vs. memory usage

#### Negative Dentry Policy
```c
static int dentry_negative_policy;
```

**Policy Options**:
- **Unlimited**: No limit on negative dentries
- **Proportional**: Limit based on system memory
- **Fixed**: Fixed limit regardless of system size
- **Adaptive**: Dynamic adjustment based on workload

### Advanced Features

#### RCU-Walk Path Resolution

#### Store-Free Path Walking
**Purpose**: Perform pathname resolution without taking locks

**RCU-Walk Benefits**:
- **High Performance**: No lock contention during lookup
- **Scalability**: Excellent multi-core scalability
- **Low Latency**: Minimal overhead for common operations
- **Robustness**: Fallback to ref-walk when needed

#### Sequence Locks for Consistency
```c
seqcount_spinlock_t d_seq;  /* Per-dentry sequence lock */
```

**Seqlock Features**:
- **Concurrent Reads**: Multiple readers without blocking
- **Consistency**: Detect concurrent modifications
- **Performance**: Optimized for read-heavy workloads
- **Memory Ordering**: Proper memory barriers

### Mount Point Integration

#### Mount Point Detection
```c
#define DCACHE_MOUNTED          0x00010000  /* Mount point */
#define DCACHE_NEED_AUTOMOUNT   0x00020000  /* Automount required */
#define DCACHE_MANAGE_TRANSIT   0x00040000  /* Transit management */
```

**Mount Integration Features**:
- **Mount Detection**: Identify mount points during traversal
- **Automount Support**: Support for automounted file systems
- **Transit Management**: Handle file system boundaries
- **Namespace Integration**: Support for mount namespaces

### Security and Access Control

#### Security Framework Integration
- **LSM Hooks**: Linux Security Module integration
- **Access Validation**: Coordinate with VFS permission checking
- **Audit Integration**: File access audit support
- **Namespace Support**: User namespace integration

### Performance Monitoring

#### Statistics Collection
```c
struct dentry_stat_t {
    long nr_dentry;        /* Total dentries */
    long nr_unused;        /* Unused dentries */
    long age_limit;        /* Age limit in seconds */
    long want_pages;       /* Requested pages */
    long nr_negative;      /* Negative dentries */
    long dummy;            /* Reserved */
};
```

**Monitoring Features**:
- **Real-time Statistics**: Current cache state
- **Performance Metrics**: Cache hit/miss ratios
- **Memory Usage**: Cache memory consumption
- **Pressure Metrics**: Memory pressure indicators

#### Pressure-Based Reclaim
```c
static int sysctl_vfs_cache_pressure __read_mostly = 100;
static int sysctl_vfs_cache_pressure_denom __read_mostly = 100;

unsigned long vfs_pressure_ratio(unsigned long val)
{
    return mult_frac(val, sysctl_vfs_cache_pressure, sysctl_vfs_cache_pressure_denom);
}
```

**Pressure Features**:
- **Tunable Pressure**: Configurable cache pressure
- **Proportional Response**: Proportional reclaim response
- **System Integration**: Integration with global memory management
- **Performance Balance**: Balance cache efficiency with memory usage

## Integration Points

### Virtual File System Integration
- **Path Resolution**: Primary interface for pathname resolution
- **Inode Management**: Master of inode cache
- **Mount Management**: Integration with mount point handling
- **Namespace Support**: File system namespace integration

### Memory Management Integration
- **Page Cache**: Coordination with file data caching
- **Slab Allocator**: Efficient memory allocation
- **LRU Management**: Integration with global LRU management
- **Memory Pressure**: Response to system memory pressure

### File System Integration
- **Generic Interface**: Common interface for all file systems
- **Custom Operations**: Support for filesystem-specific behavior
- **Revalidation**: Support for network file systems
- **Error Handling**: Consistent error handling

### Security Framework Integration
- **Access Control**: File access permission validation
- **Audit System**: File access auditing
- **Namespace Security**: Container security integration
- **Capability System**: File capability support

This comprehensive directory cache implementation provides the foundation for efficient pathname resolution in Linux, enabling fast file system access while maintaining consistency, security, and scalability across diverse file system types and workload patterns.