# kernel/exit.c - Process Termination and Wait System Implementation

## Overview

This file implements process termination and the wait family of system calls in the Linux kernel. Originally written by Linus Torvalds in 1991-1992, it provides the counterpart to `fork.c` by handling process death, resource cleanup, parent-child synchronization, and the complex mechanics of process lifecycle management.

## Historical Context

- **Original Author**: Linus Torvalds (1991-1992)
- **Evolution**: From simple exit() to complex group exit semantics
- **Modern Features**: Thread group handling, namespace cleanup, cgroup integration
- **Relationship**: Complements fork.c for complete process lifecycle

## Core Concepts

### Process Termination Types

#### Individual Thread Exit
- **Thread Death**: Single thread in multi-threaded process
- **Resource Cleanup**: Thread-specific resources only
- **Group Survival**: Process continues with remaining threads

#### Process Group Exit
- **Complete Termination**: All threads in process exit
- **Full Cleanup**: All process resources released
- **Parent Notification**: Parent process notified of death

#### Forced Termination
- **Signal-Induced**: SIGKILL, SIGTERM handling
- **Kernel Panic**: Critical system errors
- **OOM Killer**: Out-of-memory conditions

### Wait Semantics

#### Blocking Wait
- **Parent Blocking**: Parent waits for child completion
- **Status Return**: Exit code and signal information
- **Resource Reaping**: Child resources cleaned up

#### Non-Blocking Wait
- **Immediate Return**: No blocking if no children ready
- **Status Polling**: Check child status without blocking
- **Asynchronous Handling**: Event-driven child management

## Key Data Structures

### `struct wait_opts`
```c
struct wait_opts {
    enum pid_type wo_type;      /* Wait type */
    int wo_flags;               /* Wait flags */
    struct pid *wo_pid;         /* Target PID */
    struct waitid_info *wo_info; /* Wait info */
    int wo_stat;                /* Status */
    struct rusage *wo_rusage;   /* Resource usage */
    wait_queue_entry_t child_wait; /* Wait queue entry */
    int notask_error;           /* No task error */
};
```

### Exit States
- **EXIT_ZOMBIE**: Process dead but not reaped
- **EXIT_DEAD**: Process completely cleaned up
- **TASK_STOPPED**: Process stopped by signal
- **TASK_TRACED**: Process being traced

### Wait Flags
- **WNOHANG**: Non-blocking wait
- **WUNTRACED**: Report stopped children
- **WCONTINUED**: Report continued children
- **__WCLONE**: Wait for clone children
- **__WALL**: Wait for all children
- **__WNOTHREAD**: Don't wait for children of other threads

## Core Functions

### `do_exit()` - Main Exit Processing
```c
void __noreturn do_exit(long code)
```

**Purpose**: Central function that handles process termination

**Exit Sequence**:
1. **Synchronization**: Coordinate group exit if needed
2. **Signal Handling**: Set PF_EXITING, handle signals
3. **Resource Cleanup**: Release all process resources
4. **Accounting**: Update process accounting information
5. **Notification**: Notify parent and interested parties
6. **Final Cleanup**: Prepare for task structure cleanup

**Key Steps**:
```c
synchronize_group_exit(tsk, code);     // Coordinate group exit
exit_signals(tsk);                     // Signal handling cleanup
exit_mm();                             // Memory management cleanup
exit_files(tsk);                       // File descriptor cleanup
exit_fs(tsk);                          // Filesystem cleanup
exit_task_namespaces(tsk);             // Namespace cleanup
exit_notify(tsk, group_dead);          // Parent notification
```

**Critical Protections**:
- **Init Protection**: Prevents killing init process
- **Lock Validation**: Ensures no locks held at exit
- **RCU Safety**: Proper RCU cleanup sequences

### `make_task_dead()` - Signal-Induced Death
```c
void __noreturn make_task_dead(int signr)
```

**Purpose**: Handle process death from signals (SIGKILL, etc.)

**Process**:
1. **Signal Context**: Set appropriate exit code from signal
2. **Futex Cleanup**: Handle futex state for threads
3. **Exit Processing**: Call do_exit() with signal number

**Signal Handling**:
- **SIGKILL**: Unconditional termination
- **Fatal Signals**: Core dump generation
- **Group Signals**: Affect entire process group

### Exit Notification System

#### `exit_notify()` - Parent Notification
```c
static void exit_notify(struct task_struct *tsk, int group_dead)
```

**Responsibilities**:
1. **Reparenting**: Move children to new parent
2. **Zombie Creation**: Become zombie for parent to reap
3. **Signal Sending**: Send SIGCHLD to parent
4. **Ptrace Handling**: Notify ptrace parent if needed

**Reparenting Logic**:
- **Thread Death**: Children stay with thread group leader
- **Process Death**: Children reparented to init or subreaper
- **Orphan Groups**: Handle orphaned process groups

#### `__exit_signal()` - Signal Structure Cleanup
```c
static void __exit_signal(struct release_task_post *post, struct task_struct *tsk)
```

**Signal Cleanup**:
- **Timer Cleanup**: Cancel POSIX timers
- **Usage Statistics**: Accumulate CPU time and memory usage
- **Signal Queue**: Clean up pending signals
- **Group Statistics**: Update group-wide statistics

### Resource Cleanup Functions

#### `exit_mm()` - Memory Management Cleanup
```c
static void exit_mm(void)
```

**Memory Cleanup**:
1. **Address Space**: Release virtual memory address space
2. **Page Tables**: Free page table structures
3. **Memory Accounting**: Update memory usage statistics
4. **OOM Handling**: Exit OOM victim state if applicable

**Process**:
- **mm_struct Release**: Drop reference to memory descriptor
- **Page Table Cleanup**: Free page tables and VMAs
- **Memory Statistics**: Update RSS and other memory stats
- **Shared Memory**: Handle shared memory segments

#### Resource-Specific Cleanup Functions
```c
exit_files(tsk);        // Close all file descriptors
exit_fs(tsk);           // Release filesystem context
exit_sem(tsk);          // Clean up semaphore undo structures
exit_shm(tsk);          // Clean up shared memory
exit_task_namespaces(tsk); // Clean up namespace references
```

**File Descriptor Cleanup**:
- **Close All Files**: Close all open file descriptors
- **Release Tables**: Free file descriptor tables
- **Special Files**: Handle pipes, sockets, device files

**Filesystem Cleanup**:
- **Working Directory**: Release current working directory
- **Root Directory**: Release root directory reference
- **Mount Namespace**: Clean up mount namespace

## Wait System Implementation

### `do_wait()` - Main Wait Processing
```c
static long do_wait(struct wait_opts *wo)
```

**Wait Process**:
1. **Child Scanning**: Scan for eligible children
2. **State Checking**: Check child states (zombie, stopped, etc.)
3. **Blocking**: Sleep if no children ready and blocking requested
4. **Reaping**: Clean up zombie children

**Wait Queue Integration**:
- **Sleep Queue**: Add to parent's wait queue
- **Wake Events**: Child death or state change wakes parent
- **Signal Handling**: Handle interruption by signals

### Child State Handling

#### `wait_task_zombie()` - Reap Zombie Children
```c
static int wait_task_zombie(struct wait_opts *wo, struct task_struct *p)
```

**Zombie Reaping**:
1. **Status Collection**: Gather exit status and resource usage
2. **Resource Cleanup**: Release remaining resources
3. **Statistics**: Accumulate statistics in parent
4. **Task Release**: Final task_struct cleanup

**Information Gathered**:
- **Exit Code**: Process exit status
- **Resource Usage**: CPU time, memory usage, I/O statistics
- **Signal Information**: If terminated by signal

#### `wait_task_stopped()` - Handle Stopped Children
```c
static int wait_task_stopped(struct wait_opts *wo, struct task_struct *p)
```

**Stopped Child Handling**:
- **Signal Information**: Which signal stopped the child
- **Ptrace Integration**: Handle ptrace stop events
- **Status Reporting**: Return stop information to parent

#### `wait_task_continued()` - Handle Continued Children
```c
static int wait_task_continued(struct wait_opts *wo, struct task_struct *p)
```

**Continuation Handling**:
- **SIGCONT Processing**: Handle continuation from stopped state
- **Status Reporting**: Report continuation to waiting parent
- **State Transitions**: Track stopped → running transitions

### System Call Implementations

#### `SYSCALL_DEFINE1(exit, int, error_code)`
```c
SYSCALL_DEFINE1(exit, int, error_code)
```
- **Individual Exit**: Exit calling thread/process
- **Status Code**: 8-bit exit status
- **Group Behavior**: May trigger group exit

#### `SYSCALL_DEFINE1(exit_group, int, error_code)`
```c
SYSCALL_DEFINE1(exit_group, int, error_code)
```
- **Group Exit**: Terminate entire process group
- **Thread Termination**: Kill all threads in process
- **Synchronization**: Coordinate multi-thread termination

#### `SYSCALL_DEFINE4(wait4, ...)`
```c
SYSCALL_DEFINE4(wait4, pid_t, upid, int __user *, stat_addr,
                int, options, struct rusage __user *, ru)
```
- **Traditional Wait**: Classic wait4() interface
- **Resource Usage**: Optional resource usage return
- **Status Return**: Exit status and signal information

#### `SYSCALL_DEFINE5(waitid, ...)`
```c
SYSCALL_DEFINE5(waitid, int, which, pid_t, upid, struct siginfo __user *,
                infop, int, options, struct rusage __user *, ru)
```
- **Extended Wait**: More flexible wait interface
- **Detailed Information**: Extended status information
- **Signal Integration**: siginfo_t structure return

#### `SYSCALL_DEFINE3(waitpid, ...)`
```c
SYSCALL_DEFINE3(waitpid, pid_t, pid, int __user *, stat_addr, int, options)
```
- **Legacy Interface**: Compatibility with older code
- **Simplified**: Wrapper around wait4()

## Special Cases and Edge Conditions

### Group Exit Synchronization
```c
static void synchronize_group_exit(struct task_struct *tsk, long code)
```

**Multi-Thread Coordination**:
- **Group Leader**: Special handling for thread group leader
- **Signal Distribution**: Send signals to all group members
- **Exit Code**: Ensure consistent exit code across group
- **Race Prevention**: Prevent races between thread exits

### Init Process Protection
```c
if (unlikely(is_global_init(tsk)))
    panic("Attempted to kill init! exitcode=0x%08x\n", ...);
```

**System Stability**:
- **Init Immortality**: Prevent accidental init termination
- **System Panic**: Immediate panic if init dies
- **Recovery**: No recovery possible from init death

### Memory Management Owner Transfer
```c
void mm_update_next_owner(struct mm_struct *mm)
```

**Ownership Transfer**:
- **Shared Memory**: Handle shared memory ownership
- **Process Groups**: Transfer ownership within group
- **Resource Accounting**: Maintain proper resource accounting

## Performance Optimizations

### Zombie Reaping Efficiency
- **Batch Processing**: Efficient child scanning
- **Wait Queue Optimization**: Minimize unnecessary wake-ups
- **Lock Optimization**: Reduce lock contention

### Resource Cleanup Ordering
- **Dependency Handling**: Clean up in proper order
- **Reference Counting**: Efficient reference management
- **Memory Ordering**: Proper memory barriers

### Signal Handling Optimization
- **Fast Path**: Optimize common signal cases
- **Group Signals**: Efficient group signal delivery
- **Queue Management**: Optimize signal queue handling

## Security Features

### Process Isolation
- **Resource Cleanup**: Complete resource isolation after death
- **Information Leakage**: Prevent information leakage to children
- **Capability Cleanup**: Proper capability cleanup

### Signal Security
- **Permission Checks**: Verify signal sending permissions
- **Signal Filtering**: Apply signal filtering rules
- **Ptrace Security**: Secure ptrace operation handling

### Namespace Security
- **Namespace Cleanup**: Secure namespace cleanup
- **Resource Isolation**: Maintain resource isolation
- **Reference Cleanup**: Clean up namespace references

## Error Handling and Recovery

### Exit Failure Handling
- **Resource Leaks**: Prevent resource leaks on failure
- **Partial Cleanup**: Handle partial cleanup states
- **System Consistency**: Maintain system consistency

### Wait System Errors
- **EINTR**: Handle signal interruption
- **ECHILD**: No children available
- **EINVAL**: Invalid arguments

### Recovery Mechanisms
- **Orphan Handling**: Handle orphaned processes
- **Zombie Prevention**: Prevent zombie accumulation
- **Resource Recovery**: Recover leaked resources

## Integration Points

### Scheduler Integration
- **Task Removal**: Remove from scheduler queues
- **Load Balancing**: Update load balancing state
- **Statistics**: Update scheduler statistics

### Memory Management Integration
- **Page Reclaim**: Trigger page reclamation
- **Memory Accounting**: Update memory statistics
- **OOM Integration**: Interact with OOM killer

### Filesystem Integration
- **File Cleanup**: Close all open files
- **Directory References**: Release directory references
- **Mount Cleanup**: Clean up mount references

### Signal System Integration
- **Signal Delivery**: Handle pending signals
- **Signal Queues**: Clean up signal queues
- **Group Signals**: Handle group signal semantics

## Debugging and Monitoring

### Stack Usage Monitoring
```c
static void check_stack_usage(void)
```
- **Stack Overflow Detection**: Monitor stack usage
- **Statistics Collection**: Collect stack usage statistics
- **Debugging Aid**: Help debug stack issues

### Process Accounting
```c
acct_collect(code, group_dead);
```
- **Resource Tracking**: Track process resource usage
- **Audit Trail**: Maintain process audit trail
- **Statistics**: Collect system-wide statistics

### Tracing Integration
```c
trace_sched_process_exit(tsk, group_dead);
```
- **Exit Tracing**: Trace process exit events
- **Performance Analysis**: Enable performance analysis
- **Debugging**: Debug process lifecycle issues

This comprehensive implementation handles one of the most complex aspects of operating system design: safely and efficiently terminating processes while maintaining system stability, security, and proper resource cleanup. The code balances performance requirements with the need for robust error handling and security.