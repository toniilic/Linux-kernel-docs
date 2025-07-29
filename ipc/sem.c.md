# sem.c - System V Semaphores Implementation

## Overview

The `sem.c` file implements System V semaphores for Linux, providing a comprehensive inter-process synchronization mechanism. This implementation supports FIFO ordering, multiple operations per system call, UNDO functionality, and namespace isolation while maintaining excellent SMP scalability.

## File Location
- **Path**: `ipc/sem.c`
- **License**: GPL-2.0
- **Primary Authors**: 
  - Krishna Balasubramanian (1992)
  - Eric Schenk, Bruno Haible (1995)
  - Manfred Spraul (SMP-threading, lockless wakeup)
  - Davidlohr Bueso (wakeup optimizations)

## Key Features

### User-Visible Behavior
- **FIFO Ordering**: Operations processed in first-in-first-out order
- **Multiple Operations**: Single `semop()` can alter multiple semaphores
- **Time Updates**: `sem_ctime` updated on IPC_SET, SETVAL, and SETALL
- **Linux Extensions**: SEM_STAT and SEM_INFO commands
- **UNDO Support**: Process exit undo adjustments (limited to 0..SEMVMX)
- **Namespace Support**: Full namespace isolation
- **Runtime Configuration**: SEMMSL, SEMMNS, SEMOPM, SEMMNI via /proc/sys/kernel/sem
- **Statistics**: Usage reporting in /proc/sysvipc/sem

### Internal Architecture
- **Scalability**: Read-mostly global variables, RCU synchronization
- **Per-Array Locking**: Independent semaphore arrays scale perfectly
- **On-Demand Counting**: `semncnt` and `semzcnt` calculated when needed
- **Active Wakeup**: Successful operations scan and complete pending operations
- **Lockless Wakeup**: Wake-up calls performed after dropping locks
- **Waker-Does-All**: Woken tasks don't need locks or reference counting

## Core Data Structures

### sem (Individual Semaphore)
```c
struct sem {
    int semval;                     // Current semaphore value
    struct pid *sempid;             // PID of last modifier
    spinlock_t lock;                // Per-semaphore lock
    struct list_head pending_alter; // Operations that alter value
    struct list_head pending_const; // Operations that don't alter value
} ____cacheline_aligned_in_smp;
```

Each semaphore in an array, with separate pending lists for different operation types.

### sem_array (Semaphore Array)
```c
struct sem_array {
    struct kern_ipc_perm sem_perm;  // IPC permissions
    time64_t sem_ctime;             // Last change time
    struct sem *sems;               // Array of semaphores
    struct list_head pending_alter; // Global alter list
    struct list_head pending_const; // Global const list
    struct list_head list_id;       // List in namespace
    int sem_nsems;                  // Number of semaphores
    int complex_count;              // Count of complex operations
    unsigned int use_global_lock;   // Global lock usage flag
};
```

Main structure representing a semaphore array.

### sem_queue (Pending Operation)
```c
struct sem_queue {
    struct list_head list;          // Queue list node
    struct task_struct *sleeper;    // Sleeping task
    struct sem_undo *undo;          // Undo structure
    struct pid *pid;                // Process ID
    int status;                     // Operation status
    struct sembuf *sops;            // Operations array
    struct sembuf *blocking;        // Blocking operation
    int nsops;                      // Number of operations
    bool alter;                     // Whether operations alter values
    bool dupsop;                    // Duplicate operations present
};
```

Represents a pending semaphore operation waiting for completion.

### sem_undo (UNDO Operations)
```c
struct sem_undo {
    struct list_head list_proc;     // Per-process list
    struct rcu_head rcu;           // RCU callback head
    struct sem_undo_list *ulp;     // Back pointer to undo list
    struct list_head list_id;       // Per-array list
    int semid;                     // Semaphore array ID
    short *semadj;                 // UNDO adjustments array
};
```

Tracks undo adjustments for process exit cleanup.

## Scalability Design

### Lock Hierarchy
1. **Global RCU**: Protects semaphore array lookup
2. **Array-level Lock**: Protects array metadata when needed
3. **Per-Semaphore Locks**: Individual semaphore operations
4. **Wake Queue**: Lock-free wakeup mechanism

### SMP Optimization Strategies

#### Independent Array Scaling
- **Perfect Scaling**: Independent arrays have no shared state
- **Cache Line Optimization**: Per-semaphore locks prevent false sharing
- **RCU Protection**: Lookup operations don't require locks

#### Smart Locking
- **Global vs Local**: Switches between global and per-semaphore locking
- **Complex Operation Detection**: Uses global lock for complex multi-semaphore operations
- **Lock Coalescing**: Reduces lock acquire/release cycles

#### Lockless Wakeup
- **Deferred Wakeup**: Wake-up calls happen after dropping all locks
- **Wake Queue Framework**: Batches multiple wakeups
- **Zero Copy Wakeup**: Woken tasks don't touch semaphore array

## Operation Processing

### semop() Flow
1. **Validation**: Check permissions and parameters
2. **Lock Acquisition**: Acquire appropriate locks (global or per-semaphore)
3. **Immediate Check**: Try to complete operation immediately
4. **Queue or Complete**: Either queue for later or complete now
5. **Wakeup Scan**: If successful, scan for other completable operations
6. **Lock Release**: Release locks and perform wakeups

### FIFO Guarantee
- **Dual Lists**: Per-array and per-semaphore pending lists
- **Ordered Processing**: Operations processed in submission order
- **Starvation Prevention**: FIFO prevents indefinite blocking

### Complex Operations
Operations affecting multiple semaphores use global array lock to ensure atomicity:
- **Detection**: Counted during operation parsing
- **Lock Selection**: Global lock chosen for complex operations
- **Atomicity**: All-or-nothing semantics maintained

## UNDO Mechanism

### Process Exit Cleanup
- **Lazy Allocation**: UNDO structures allocated on first use
- **Per-Process Tracking**: Each process has undo list
- **Exit Processing**: All undo adjustments applied at process exit
- **Namespace Aware**: UNDO operations respect namespace boundaries

### Undo Value Management
- **Range Limiting**: Values clamped to 0..SEMVMX range
- **Overflow Handling**: Prevents integer overflow in undo values
- **Memory Efficiency**: Shared undo structures where possible

## Memory Management

### Allocation Strategies
- **SLAB Caches**: Dedicated caches for frequent structures
- **RCU Freeing**: Safe memory reclamation under RCU
- **Reference Counting**: Prevents premature structure freeing

### Cache Line Optimization
- **Structure Alignment**: Critical structures cache-line aligned
- **False Sharing Prevention**: Per-CPU and per-semaphore data separated
- **Memory Locality**: Related data grouped for better cache usage

## Synchronization Primitives

### RCU Usage
- **Array Lookup**: RCU protects semaphore array discovery
- **Removal Safety**: Arrays can be safely accessed even during removal
- **Grace Periods**: Proper grace period management for structure freeing

### Spinlock Strategy
- **Fine-Grained Locking**: Per-semaphore locks when possible
- **Global Fallback**: Global lock for complex operations
- **Lock Ordering**: Consistent ordering prevents deadlocks

### Wake Queue Framework
- **Batch Processing**: Multiple tasks woken efficiently
- **Lock-Free**: Wakeups happen without holding semaphore locks
- **Task Safety**: Reference counting prevents use-after-free

## Performance Characteristics

### Best Case Scenarios
- **Independent Arrays**: Perfect SMP scaling
- **Simple Operations**: Single semaphore, immediate completion
- **No Contention**: Lock-free fast paths

### Worst Case Scenarios
- **Complex Operations**: O(N²) wakeup scanning
- **Heavy Contention**: Cache line thrashing on array locks
- **Large Arrays**: Memory pressure from large semaphore arrays

### Optimization Techniques
- **Adaptive Locking**: Switches between global and fine-grained locks
- **Lazy Evaluation**: Defers expensive operations when possible
- **Batch Operations**: Groups related operations for efficiency

## Error Handling and Edge Cases

### Resource Limits
- **Array Limits**: Maximum arrays per namespace (SEMMNI)
- **Semaphore Limits**: Maximum semaphores per array (SEMMSL)
- **Operation Limits**: Maximum operations per call (SEMOPM)
- **Total Limits**: System-wide semaphore limit (SEMMNS)

### Signal Handling
- **Interruptible Waits**: Operations can be interrupted by signals
- **Restart Logic**: Proper handling of interrupted system calls
- **UNDO Consistency**: Signal interruption doesn't corrupt undo state

### Race Condition Prevention
- **RCU Grace Periods**: Prevent access to freed structures
- **Reference Counting**: Ensures structure lifetime management
- **Memory Barriers**: Proper ordering for lockless operations

## Namespace Integration

### IPC Namespace Support
- **Isolation**: Complete isolation between namespaces
- **Resource Tracking**: Per-namespace resource accounting
- **Migration Safety**: Proper handling during namespace transitions

### Permission Model
- **Standard IPC**: Uses kern_ipc_perm for permission checking
- **User Namespace**: Integrates with user namespace UID/GID mapping
- **Capability Checking**: Proper capability verification

## Debugging and Monitoring

### Procfs Interface
- **/proc/sysvipc/sem**: Current semaphore status
- **/proc/sys/kernel/sem**: Runtime configuration
- **Statistics**: Usage and performance metrics

### Audit Integration
- **Operation Logging**: Security-relevant operations logged
- **Permission Changes**: Audit trail for permission modifications
- **Namespace Events**: Namespace creation/destruction tracking

## Compatibility and Standards

### POSIX Compliance
- **Standard Operations**: semget, semop, semctl
- **Error Codes**: Standard errno values
- **Behavior Matching**: Matches POSIX semaphore semantics

### Linux Extensions
- **SEM_STAT**: Extended status information
- **SEM_INFO**: System-wide information
- **Namespace Support**: Linux-specific namespace isolation

This implementation represents a highly optimized and scalable semaphore system that balances POSIX compliance with Linux-specific performance and security features, making it suitable for high-performance computing environments while maintaining compatibility with existing applications.