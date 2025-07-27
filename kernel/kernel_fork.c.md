# kernel/fork.c - Process Creation and Cloning Implementation

## Overview

This file contains the core implementation of process creation in the Linux kernel, including the fork(), clone(), clone3(), and vfork() system calls. Originally written by Linus Torvalds in 1991-1992, it remains one of the most critical and complex parts of the kernel, handling process duplication, resource management, and the intricate details of creating new execution contexts.

## Historical Context

- **Original Author**: Linus Torvalds (1991-1992)
- **Evolution**: From simple fork() to complex clone() with namespaces
- **Modern Extensions**: clone3() system call for extensibility
- **Memory Management**: Close integration with mm/memory.c for copy-on-write

## Core Concepts

### Process Creation Methods

#### Traditional Unix Fork
- **fork()**: Creates exact copy of parent process
- **Memory**: Copy-on-write semantics for efficiency
- **Return Value**: Child gets 0, parent gets child PID

#### Modern Clone Interface
- **clone()**: Selective sharing of process attributes
- **Namespaces**: Support for containerization
- **Threads**: POSIX thread creation support

#### Optimized Variants
- **vfork()**: Optimized for exec() scenarios
- **clone3()**: Extended interface with more options

## Key Data Structures

### `struct kernel_clone_args`
```c
struct kernel_clone_args {
    u64 flags;              /* Clone flags */
    int __user *pidfd;      /* PID file descriptor */
    int __user *child_tid;  /* Child thread ID */
    int __user *parent_tid; /* Parent thread ID */
    int exit_signal;        /* Signal on child exit */
    unsigned long stack;    /* Stack pointer */
    unsigned long stack_size; /* Stack size */
    unsigned long tls;      /* Thread-local storage */
    pid_t *set_tid;        /* Set specific TIDs */
    size_t set_tid_size;   /* Number of TIDs to set */
    int cgroup;            /* Target cgroup */
    int io_thread;         /* IO thread flag */
    int kthread;           /* Kernel thread flag */
    int idle;              /* Idle task flag */
    int (*fn)(void *);     /* Thread function */
    void *fn_arg;          /* Thread function argument */
    struct cgroup *cgrp;   /* Cgroup context */
    struct css_set *cset;  /* CSS set */
};
```

### Clone Flags (Partial List)
- **CLONE_VM**: Share virtual memory
- **CLONE_FS**: Share filesystem information  
- **CLONE_FILES**: Share file descriptor table
- **CLONE_SIGHAND**: Share signal handlers
- **CLONE_THREAD**: Create thread (share PID)
- **CLONE_NEWNS**: New mount namespace
- **CLONE_NEWPID**: New PID namespace
- **CLONE_NEWNET**: New network namespace
- **CLONE_VFORK**: vfork() semantics
- **CLONE_PIDFD**: Return PID file descriptor

## Core Functions

### `copy_process()` - Central Process Duplication
```c
struct task_struct *copy_process(struct pid *pid, int trace, int node,
                                struct kernel_clone_args *args)
```

**Purpose**: Core function that creates a new task by copying/sharing resources from parent

**Key Steps**:
1. **Validation**: Verify clone flags compatibility
2. **Task Allocation**: Allocate new task_struct
3. **Resource Copying**: Copy/share various process resources
4. **Security Setup**: Apply security contexts and capabilities
5. **Scheduling Setup**: Initialize scheduler state
6. **Namespace Setup**: Handle namespace creation/sharing
7. **Final Setup**: Complete process initialization

**Resource Copying Functions**:
- `copy_mm()`: Memory management
- `copy_fs()`: Filesystem context
- `copy_files()`: File descriptor table
- `copy_sighand()`: Signal handlers
- `copy_signal()`: Signal state
- `copy_semundo()`: Semaphore undo
- `copy_namespaces()`: Namespace contexts
- `copy_io()`: I/O context

**Error Handling**: Extensive cleanup on failure with `bad_fork_*` labels

### `kernel_clone()` - Main Cloning Interface  
```c
pid_t kernel_clone(struct kernel_clone_args *args)
```

**Purpose**: Primary internal interface for process creation

**Process**:
1. **Argument Validation**: Verify clone arguments
2. **Tracing Setup**: Handle ptrace requirements
3. **Process Creation**: Call copy_process()
4. **Parent Notification**: Update parent_tid if requested
5. **vfork Handling**: Handle vfork completion
6. **Wake Child**: Make child process runnable

**Special Handling**:
- **PIDFD Support**: Create PID file descriptors
- **vfork Synchronization**: Parent waits for child exec/exit
- **LRU Generation**: Handle memory management optimization

### System Call Implementations

#### `SYSCALL_DEFINE0(fork)`
```c
SYSCALL_DEFINE0(fork)
```
- **Traditional fork**: Complete process duplication
- **Flags**: No sharing flags set
- **Return**: Child PID to parent, 0 to child

#### `SYSCALL_DEFINE0(vfork)` 
```c
SYSCALL_DEFINE0(vfork)
```
- **Optimized fork**: For immediate exec() scenarios
- **Sharing**: Shares memory until exec() or exit()
- **Synchronization**: Parent blocks until child releases

#### `SYSCALL_DEFINE5(clone, ...)`
```c
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, ...)
```
- **Flexible creation**: Selective resource sharing
- **Threading Support**: Used by pthread libraries
- **Namespace Support**: Container creation

#### `SYSCALL_DEFINE2(clone3, ...)`
```c
SYSCALL_DEFINE2(clone3, struct clone_args __user *, uargs, size_t, size)
```
- **Extended interface**: More flexible than clone()
- **Extensible**: Forward-compatible design
- **New Features**: PID file descriptors, cgroup selection

### Memory and Stack Management

#### Thread Stack Allocation
```c
static int alloc_thread_stack_node(struct task_struct *tsk, int node)
static void free_thread_stack(struct task_struct *tsk)
```

**Stack Management Features**:
- **NUMA Awareness**: Allocate on appropriate node
- **Stack Caching**: Reuse stacks for performance
- **Guard Pages**: Detect stack overflow
- **Memory Control**: Charge to appropriate cgroup

**Stack Allocation Strategies**:
1. **Cached Stacks**: Reuse from per-CPU cache
2. **Direct Allocation**: Allocate new stack
3. **VMAP Stacks**: Use virtual memory mapping
4. **Memory Charging**: Account for memory usage

#### Stack Security Features
- **Stack Canaries**: Detect buffer overflows
- **Guard Pages**: Prevent stack overflow
- **Randomization**: ASLR for stack placement
- **Leak Prevention**: Clear sensitive data

### Resource Copying Details

#### Memory Management Copy (`copy_mm`)
```c
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
```

**CLONE_VM Behavior**:
- **Shared Memory**: Both processes share same mm_struct
- **Threading**: Used for POSIX threads
- **Synchronization**: Shared page tables and VMAs

**Non-CLONE_VM Behavior**:
- **Copy-on-Write**: Duplicate but share pages initially
- **Independent Address Space**: Separate virtual memory
- **Fork Optimization**: Pages copied only when modified

#### File Descriptor Copy (`copy_files`)
```c
static int copy_files(unsigned long clone_flags, struct task_struct *tsk,
                     int no_files)
```

**CLONE_FILES Behavior**:
- **Shared Table**: Both processes share same file table
- **Reference Counting**: Automatic cleanup
- **Synchronization**: Changes visible to both processes

**Non-CLONE_FILES Behavior**:
- **Duplicate Table**: Independent file descriptor spaces
- **FD_CLOEXEC**: Proper handling of close-on-exec
- **Resource Limits**: Independent RLIMIT_NOFILE

#### Signal Handler Copy (`copy_sighand`)
```c
static int copy_sighand(unsigned long clone_flags, struct task_struct *tsk)
```

**CLONE_SIGHAND Behavior**:
- **Shared Handlers**: Both processes share signal handlers
- **Threading Requirement**: Must also use CLONE_VM
- **Handler Changes**: Visible to both processes

**Signal State Copy (`copy_signal`)**:
- **Process Groups**: Proper PID/PGID setup
- **Signal Masks**: Independent signal blocking
- **Pending Signals**: Separate signal queues

### Namespace and Security

#### Namespace Copying (`copy_namespaces`)
```c
static int copy_namespaces(unsigned long clone_flags, struct task_struct *tsk)
```

**Namespace Types**:
- **Mount (CLONE_NEWNS)**: Independent filesystem view
- **PID (CLONE_NEWPID)**: Separate PID number space
- **Network (CLONE_NEWNET)**: Independent network stack
- **User (CLONE_NEWUSER)**: Separate user/group IDs
- **IPC**: Independent System V IPC
- **UTS**: Independent hostname/domain
- **Cgroup**: Independent cgroup view

#### Security Context Setup
```c
retval = security_task_alloc(p, clone_flags);
```

**Security Features**:
- **LSM Integration**: SELinux, AppArmor, etc.
- **Capabilities**: Proper capability inheritance
- **Credentials**: User/group ID setup
- **Audit Trail**: Process creation logging

### Process Lifecycle Management

#### Process Birth
1. **Allocation**: Create task_struct
2. **Resource Setup**: Copy/share resources
3. **Security**: Apply security contexts
4. **Scheduling**: Make process runnable
5. **Notification**: Inform parent of creation

#### Parent-Child Relationships
- **Process Groups**: PGID management
- **Sessions**: Session ID handling
- **Orphan Processes**: Reparenting to init
- **Wait Semantics**: Parent waiting for child

#### vfork Special Handling
```c
static int wait_for_vfork_done(struct task_struct *child,
                              struct completion *vfork)
```

**vfork Optimization**:
- **Memory Sharing**: Temporary sharing until exec()
- **Parent Blocking**: Parent waits for child
- **Exec Notification**: Child signals when ready
- **Efficiency**: Avoids unnecessary copying

## Process Creation Flow

### Standard Fork Flow
1. **System Call Entry**: fork() system call
2. **Argument Setup**: Prepare kernel_clone_args
3. **Validation**: Check capabilities and limits
4. **Process Creation**: Call copy_process()
5. **Resource Copying**: Copy/share all resources
6. **Security Setup**: Apply security policies
7. **Process Activation**: Make child runnable
8. **Return**: Return PID to parent, 0 to child

### Clone Flow with Namespace Creation
1. **Argument Parsing**: Parse complex clone flags
2. **Namespace Validation**: Check namespace permissions
3. **Resource Planning**: Determine what to copy/share
4. **Security Checks**: Verify security constraints
5. **Namespace Creation**: Create new namespaces
6. **Process Creation**: Create task with new context
7. **Initialization**: Initialize namespace-specific state
8. **Activation**: Make process runnable in new context

### Thread Creation Flow
1. **Threading Validation**: Verify CLONE_VM + CLONE_SIGHAND
2. **Shared Resource Setup**: Share memory and signal handlers
3. **Thread-Specific Setup**: TLS, stack, thread ID
4. **Signal Handling**: Configure thread signal semantics
5. **Scheduler Integration**: Add to thread group
6. **Activation**: Start thread execution

## Performance Optimizations

### Copy-on-Write (COW)
- **Memory Efficiency**: Share pages until modification
- **Write Protection**: Hardware-assisted copy detection
- **Page Faulting**: Copy pages on first write
- **Reference Counting**: Track page sharing

### Stack Caching
- **Per-CPU Caches**: Reuse thread stacks
- **NUMA Optimization**: Allocate on local node
- **Batch Operations**: Efficient stack management
- **Memory Pressure**: Adaptive cache sizing

### vfork Optimization
- **Memory Sharing**: Avoid copy until exec()
- **Parent Blocking**: Efficient synchronization
- **Quick Path**: Optimized for exec() pattern
- **Compatibility**: Maintains POSIX semantics

## Security Features

### Process Isolation
- **Address Space**: Independent virtual memory
- **File Descriptors**: Separate file tables
- **Signal Handling**: Independent signal processing
- **Capabilities**: Proper privilege isolation

### Namespace Security
- **User Namespaces**: UID/GID mapping
- **Network Isolation**: Separate network stacks
- **Mount Isolation**: Independent filesystem views
- **PID Isolation**: Separate process ID spaces

### Resource Limits
- **RLIMIT_NPROC**: Process count limits
- **Memory Limits**: Cgroup memory constraints
- **CPU Limits**: Scheduling constraints
- **File Limits**: Open file restrictions

## Error Handling and Cleanup

### Failure Recovery
- **Resource Cleanup**: Proper cleanup on failure
- **Partial State**: Handle partially initialized processes
- **Memory Leaks**: Prevent resource leaks
- **Security State**: Clean security contexts

### Error Propagation
- **Clear Error Codes**: Specific error returns
- **Cleanup Labels**: Structured cleanup paths
- **State Consistency**: Maintain kernel consistency
- **User Notification**: Proper error reporting

## Integration Points

### Scheduler Integration
- **Task Queues**: Add new tasks to scheduler
- **CPU Affinity**: Inherit or set CPU affinity
- **Priority**: Set initial scheduling priority
- **Load Balancing**: Distribute across CPUs

### Memory Management
- **Page Tables**: Set up virtual memory
- **COW Setup**: Configure copy-on-write
- **Memory Accounting**: Track memory usage
- **NUMA**: Memory placement optimization

### Filesystem Integration
- **Working Directory**: Inherit or set CWD
- **Root Directory**: Handle chroot environments
- **File Descriptors**: Manage FD inheritance
- **Mount Namespaces**: Handle filesystem views

### Networking Integration
- **Network Namespaces**: Separate network views
- **Socket Inheritance**: Handle socket sharing
- **Network Configuration**: Set up network context
- **Security Contexts**: Network security labels

This implementation represents one of the most sophisticated process creation systems in any operating system, balancing performance, security, and flexibility while supporting modern containerization and threading requirements.