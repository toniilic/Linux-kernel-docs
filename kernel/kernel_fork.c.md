# kernel/fork.c - Process Creation and Cloning Implementation

## Overview

This file contains the core implementation of process creation in the Linux kernel, including the fork(), clone(), clone3(), and vfork() system calls. Originally written by Linus Torvalds in 1991-1992, it remains one of the most critical and complex parts of the kernel, handling process duplication, resource management, and the intricate details of creating new execution contexts.

This comprehensive documentation analyzes the implementation through five specialized perspectives:
1. **Core fork/clone algorithms and task creation**
2. **Memory management during fork (COW and address space)**  
3. **Thread creation and thread group management**
4. **Resource inheritance and security context copying**
5. **Scheduler integration and performance optimizations**

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

## Agent 1: Core Fork/Clone Algorithms and Task Creation

### Task Structure Duplication

#### `dup_task_struct()` - Foundation of Process Creation
```c
static struct task_struct *dup_task_struct(struct task_struct *orig, int node)
```

**Core Algorithm**:
1. **Memory Allocation**: Allocate new task_struct on specified NUMA node
2. **Stack Allocation**: Allocate kernel stack with guard pages
3. **Structure Copy**: Copy parent task_struct to child
4. **Stack Initialization**: Set up stack pointer and thread_info
5. **Reference Counting**: Initialize refcount and usage counters
6. **Security Context**: Copy basic security attributes

**Critical Implementation Details**:
- **NUMA Optimization**: Uses `alloc_task_struct_node(node)` for locality
- **Stack Protection**: Implements stack canaries and guard pages
- **Thread Info**: Copies thread_info structure for architecture-specific state
- **Memory Accounting**: Tracks memory usage in cgroups

#### Task Allocation Flow
```c
// Simplified allocation sequence
p = alloc_task_struct_node(node);          // Allocate task structure
if (!p) return NULL;
thread_stack = alloc_thread_stack_node(p, node);  // Allocate stack
if (!thread_stack) {
    free_task_struct(p);
    return NULL;
}
setup_thread_stack(p, orig);               // Initialize stack
```

### PID Allocation and Management

#### `alloc_pid()` - Process Identifier Assignment
```c
struct pid *alloc_pid(struct pid_namespace *ns, pid_t *set_tid, size_t set_tid_size)
```

**PID Allocation Strategy**:
1. **Namespace Traversal**: Allocate PIDs in all ancestor namespaces
2. **Collision Detection**: Use bitmap for efficient PID tracking
3. **Wrap-around Handling**: Manage PID space exhaustion
4. **Security Checks**: Validate PID assignment permissions

**Multi-namespace PID Management**:
```c
for (i = ns->level; i >= 0; i--) {
    nr = idr_alloc_cyclic(&tmp->idr, NULL, pid_min, pid_max, GFP_ATOMIC);
    if (nr < 0) {
        retval = (nr == -ENOSPC) ? -EAGAIN : nr;
        goto out_free;
    }
    pid->numbers[i].nr = nr;
    pid->numbers[i].ns = tmp;
    tmp = tmp->parent;
}
```

### Clone Flag Validation

#### Compatibility Matrix
| Flag Combination | Valid | Reason |
|------------------|-------|--------|
| CLONE_THREAD + CLONE_SIGHAND | ✓ | Threads must share signal handlers |
| CLONE_SIGHAND + CLONE_VM | ✓ | Signal handlers require shared memory |
| CLONE_NEWNS + CLONE_FS | ✗ | Conflicting filesystem semantics |
| CLONE_NEWUSER + CLONE_FS | ✗ | Security isolation violation |
| CLONE_THREAD + CLONE_NEWPID | ✗ | Threads can't have different PID namespace |

#### Validation Logic
```c
// Thread groups must share signals and VM
if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))
    return ERR_PTR(-EINVAL);

// Shared signal handlers require shared VM
if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
    return ERR_PTR(-EINVAL);

// Namespace isolation checks
if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS))
    return ERR_PTR(-EINVAL);
```

### Process State Initialization

#### Critical State Setup in `copy_process()`
```c
// Initialize basic task state
p->flags &= ~(PF_SUPERPRIV | PF_WQ_WORKER | PF_IDLE);
p->flags |= PF_FORKNOEXEC;  // Mark as forked but not exec'd

// Initialize lists and locks
INIT_LIST_HEAD(&p->children);
INIT_LIST_HEAD(&p->sibling);
spin_lock_init(&p->alloc_lock);

// Initialize signal handling
init_sigpending(&p->pending);

// Initialize timing
p->utime = p->stime = p->gtime = 0;
prev_cputime_init(&p->prev_cputime);

// Initialize I/O accounting
task_io_accounting_init(&p->ioac);
acct_clear_integrals(p);
```

## Agent 2: Memory Management During Fork (COW and Address Space)

### Copy-on-Write Implementation

#### Memory Descriptor Handling in `copy_mm()`
```c
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
    struct mm_struct *mm, *oldmm;
    
    tsk->min_flt = tsk->maj_flt = 0;
    tsk->nvcsw = tsk->nivcsw = 0;
    
    if (clone_flags & CLONE_VM) {
        mmget(current->mm);  // Share memory space
        tsk->mm = current->mm;
        return 0;
    }
    
    // Create new memory descriptor
    mm = dup_mm(tsk, current->mm);
    if (!mm)
        return -ENOMEM;
        
    tsk->mm = mm;
    return 0;
}
```

#### Address Space Duplication (`dup_mm`)
```c
static struct mm_struct *dup_mm(struct task_struct *tsk, struct mm_struct *oldmm)
{
    struct mm_struct *mm;
    int err;
    
    mm = allocate_mm();  // Allocate new mm_struct
    if (!mm)
        return NULL;
        
    memcpy(mm, oldmm, sizeof(*mm));  // Copy basic structure
    
    if (!mm_init(mm, tsk, mm->user_ns)) {
        goto fail_nomem;
    }
    
    err = dup_mmap(mm, oldmm);  // Duplicate memory mappings
    if (err)
        goto free_pt;
        
    return mm;
}
```

#### VMA Duplication and COW Setup
```c
int dup_mmap(struct mm_struct *mm, struct mm_struct *oldmm)
{
    struct vm_area_struct *mpnt, *tmp;
    int retval;
    
    flush_cache_dup_mm(oldmm);
    uprobe_dup_mmap(oldmm, mm);
    
    VMA_ITERATOR(old_vmi, oldmm, 0);
    VMA_ITERATOR(vmi, mm, 0);
    
    for_each_vma(old_vmi, mpnt) {
        if (mpnt->vm_flags & VM_DONTCOPY)
            continue;
            
        tmp = vm_area_dup(mpnt);  // Duplicate VMA
        if (!tmp)
            goto fail_nomem;
            
        retval = vma_dup_policy(mpnt, tmp);  // Copy memory policy
        if (retval)
            goto fail_nomem_policy;
            
        tmp->vm_mm = mm;
        retval = dup_userfaultfd(tmp, &uf);  // Handle userfaultfd
        if (retval)
            goto fail_nomem_anon_vma_fork;
            
        if (tmp->vm_flags & VM_SHARED)
            mapping_dup_flags(tmp->vm_file->f_mapping);
            
        // Set up copy-on-write
        if (!(tmp->vm_flags & VM_WIPEONFORK))
            retval = copy_page_range(tmp, mpnt);  // COW setup
            
        if (retval)
            goto fail_nomem_policy;
            
        if (vma_iter_bulk_store(&vmi, tmp))
            goto fail_nomem_policy;
            
        mm->map_count++;
    }
    
    retval = arch_dup_mmap(oldmm, mm);
    return retval;
}
```

### Page Table Copy-on-Write Setup

#### `copy_page_range()` - Core COW Implementation
```c
int copy_page_range(struct vm_area_struct *dst_vma, struct vm_area_struct *src_vma)
{
    pgd_t *src_pgd, *dst_pgd;
    unsigned long next;
    unsigned long addr = src_vma->vm_start;
    unsigned long end = src_vma->vm_end;
    struct mmu_notifier_range range;
    bool is_cow;
    int ret;
    
    // Skip special VMAs
    if (!(src_vma->vm_flags & (VM_HUGETLB | VM_PFNMAP | VM_MIXEDMAP))) {
        if (!src_vma->anon_vma)
            return 0;
    }
    
    if (is_vm_hugetlb_page(src_vma))
        return copy_hugetlb_page_range(dst_mm, src_mm, src_vma);
        
    if (unlikely(src_vma->vm_flags & VM_PFNMAP)) {
        ret = track_pfn_copy(src_vma);
        if (ret)
            return ret;
    }
    
    // Set up for copy-on-write
    is_cow = is_cow_mapping(src_vma->vm_flags);
    if (is_cow) {
        mmu_notifier_range_init(&range, MMU_NOTIFY_PROTECTION_PAGE,
                              0, src_vma, src_mm, addr, end);
        mmu_notifier_invalidate_range_start(&range);
        /* 
         * Disabling preemption is not needed for the write side, as
         * the read side doesn't spin, but goes to the mmap_lock.
         */
        raw_write_seqcount_begin(&src_mm->write_protect_seq);
    }
    
    // Copy page tables with COW setup
    ret = 0;
    dst_pgd = pgd_offset(dst_mm, addr);
    src_pgd = pgd_offset(src_mm, addr);
    do {
        next = pgd_addr_end(addr, end);
        if (pgd_none_or_clear_bad(src_pgd))
            continue;
        if (unlikely(copy_p4d_range(dst_mm, src_mm, dst_pgd, src_pgd,
                                  src_vma, addr, next))) {
            ret = -ENOMEM;
            break;
        }
    } while (dst_pgd++, src_pgd++, addr = next, addr != end);
    
    if (is_cow) {
        raw_write_seqcount_end(&src_mm->write_protect_seq);
        mmu_notifier_invalidate_range_end(&range);
    }
    
    return ret;
}
```

#### COW Page Fault Handling
When a write occurs to a COW page:
1. **Page Fault**: Hardware generates page fault on write to read-only page
2. **COW Detection**: Kernel identifies this as COW fault
3. **Page Allocation**: Allocate new page for writing process
4. **Page Copy**: Copy content from shared page to new page
5. **PTE Update**: Update page table entry to point to new writable page
6. **TLB Flush**: Invalidate TLB entries for the page

### Memory Accounting and Limits

#### Memory Cgroup Integration
```c
// Memory charging for new process
if (memcg_kmem_enabled() && !IS_ENABLED(CONFIG_SLOB)) {
    ret = memcg_alloc_page_obj_cgroups(page, mm, gfp);
    if (ret)
        return ret;
}

// Memory usage tracking
mm->total_vm += len >> PAGE_SHIFT;
mm->data_vm += len >> PAGE_SHIFT;

// RSS (Resident Set Size) tracking
add_mm_counter(mm, MM_ANONPAGES, nr_pages);
add_mm_counter(mm, MM_FILEPAGES, nr_pages);
```

## Agent 3: Thread Creation and Thread Group Management

### Thread vs Process Creation

#### Thread Creation Flags
```c
#define CLONE_THREAD_FLAGS (CLONE_VM | CLONE_FS | CLONE_FILES | \
                           CLONE_SIGHAND | CLONE_THREAD | \
                           CLONE_SETTLS | CLONE_PARENT_SETTID | \
                           CLONE_CHILD_CLEARTID)
```

#### Thread-Specific Setup in `copy_process()`
```c
if (clone_flags & CLONE_THREAD) {
    // Validate thread creation
    if ((clone_flags & (CLONE_NEWUSER | CLONE_NEWPID)) ||
        (task_active_pid_ns(current) != nsp->pid_ns_for_children))
        return ERR_PTR(-EINVAL);
        
    // Share thread group leader
    p->group_leader = current->group_leader;
    
    // Add to thread group
    list_add_tail_rcu(&p->thread_node, &p->signal->thread_head);
    
    // Share process ID
    p->pid = current->pid;
    p->tgid = current->tgid;
}
```

### Thread Group Management

#### Thread Group Leader Setup
```c
if (thread_group_leader(p)) {
    // Set up as thread group leader
    p->signal->leader_pid = pid;
    p->signal->tty = tty_kref_get(current->signal->tty);
    
    // Initialize thread group accounting
    p->signal->cutime = p->signal->cstime = 0;
    p->signal->min_flt = p->signal->maj_flt = 0;
    p->signal->nvcsw = p->signal->nivcsw = 0;
    
    // Set up signal handling for group
    p->signal->shared_pending.signal = delayed.signal;
    p->signal->group_exec_id = current->signal->group_exec_id;
}
```

#### Thread ID (TID) Management
```c
// Thread-specific PID assignment
if (clone_flags & CLONE_THREAD) {
    // Threads share the same PID but have unique TIDs
    p->pid = current->pid;
    p->thread_pid = get_pid(pid);  // Unique thread PID structure
} else {
    // New process gets new PID
    if (pid != &init_struct_pid) {
        p->pid = pid_nr(pid);
        p->thread_pid = pid;
    }
}
```

### Signal Handling in Thread Groups

#### Shared Signal Handler Setup
```c
static int copy_sighand(unsigned long clone_flags, struct task_struct *tsk)
{
    struct sighand_struct *sig;
    
    if (clone_flags & CLONE_SIGHAND) {
        // Share signal handlers
        refcount_inc(&current->sighand->count);
        return 0;
    }
    
    // Create new signal handler table
    sig = kmem_cache_alloc(sighand_cachep, GFP_KERNEL);
    if (!sig)
        return -ENOMEM;
        
    refcount_set(&sig->count, 1);
    spin_lock_init(&sig->siglock);
    memcpy(sig->action, current->sighand->action, sizeof(sig->action));
    
    // Handle CLONE_CLEAR_SIGHAND
    if (clone_flags & CLONE_CLEAR_SIGHAND) {
        for (i = 0; i < _NSIG; i++) {
            if (sig->action[i].sa.sa_handler != SIG_DFL &&
                sig->action[i].sa.sa_handler != SIG_IGN)
                sig->action[i].sa.sa_handler = SIG_DFL;
        }
    }
    
    tsk->sighand = sig;
    return 0;
}
```

#### Thread Group Signal Delivery
```c
// Signal delivery to thread groups
static int copy_signal(unsigned long clone_flags, struct task_struct *tsk)
{
    struct signal_struct *sig;
    
    if (clone_flags & CLONE_THREAD)
        return 0;  // Threads share signal struct
        
    // Create new signal struct for new process
    sig = kmem_cache_zalloc(signal_cachep, GFP_KERNEL);
    if (!sig)
        return -ENOMEM;
        
    sig->nr_threads = 1;
    atomic_set(&sig->live, 1);
    refcount_set(&sig->sigcnt, 1);
    
    // Initialize thread group head
    INIT_LIST_HEAD(&sig->thread_head);
    
    // Initialize signal queues
    init_sigpending(&sig->shared_pending);
    INIT_HLIST_HEAD(&sig->multiprocess);
    
    // Set up signal statistics
    sig->utime = sig->stime = sig->cutime = sig->cstime = 0;
    sig->gtime = 0;
    sig->cgtime = 0;
    sig->min_flt = sig->maj_flt = 0;
    sig->nvcsw = sig->nivcsw = 0;
    
    tsk->signal = sig;
    return 0;
}
```

### Thread Local Storage (TLS)

#### TLS Setup for Threads
```c
if (clone_flags & CLONE_SETTLS) {
    retval = set_new_tls(p, args->tls);
    if (retval)
        goto bad_fork_cleanup_io;
}
```

#### Architecture-Specific TLS Implementation
```c
// x86_64 TLS setup example
static int set_new_tls(struct task_struct *p, unsigned long tls)
{
    struct user_desc __user *utls = (struct user_desc __user *)tls;
    struct user_desc tls_info;
    
    if (copy_from_user(&tls_info, utls, sizeof(tls_info)))
        return -EFAULT;
        
    return do_set_thread_area(p, -1, &tls_info, 0);
}
```

## Agent 4: Resource Inheritance and Security Context Copying

### Credential Copying and Security Context

#### Credential Duplication (`copy_creds`)
```c
int copy_creds(struct task_struct *p, unsigned long clone_flags)
{
    struct cred *new;
    int ret;
    
    if (
#ifdef CONFIG_KEYS
        !p->cred->thread_keyring &&
#endif
        clone_flags & CLONE_THREAD
    ) {
        p->real_cred = get_cred(p->cred);
        get_cred(p->cred);
        alter_cred_subscribers(p->cred, 2);
        kdebug("share_creds(%p{%d,%d})",
               p->cred, atomic_read(&p->cred->usage),
               read_cred_subscribers(p->cred));
        inc_rlimit_ucounts(task_ucounts(p), UCOUNT_RLIMIT_NPROC, 1);
        return 0;
    }
    
    new = prepare_creds();
    if (!new)
        return -ENOMEM;
        
    if (clone_flags & CLONE_NEWUSER) {
        ret = create_user_ns(new);
        if (ret < 0)
            goto error_put;
    }
    
#ifdef CONFIG_KEYS
    ret = copy_thread_group_keys(p);
    if (ret < 0)
        goto error_put;
#endif
    
    p->cred = p->real_cred = get_cred(new);
    alter_cred_subscribers(new, 2);
    validate_creds(new);
    inc_rlimit_ucounts(task_ucounts(p), UCOUNT_RLIMIT_NPROC, 1);
    return 0;
    
error_put:
    put_cred(new);
    return ret;
}
```

#### Linux Security Module (LSM) Integration
```c
// Security task allocation
retval = security_task_alloc(p, clone_flags);
if (retval)
    goto bad_fork_cleanup_policy;
    
// Security context setup in security_task_alloc()
int security_task_alloc(struct task_struct *task, unsigned long clone_flags)
{
    int rc = lsm_task_alloc(task);
    
    if (rc)
        return rc;
        
    rc = call_int_hook(task_alloc, 0, task, clone_flags);
    if (unlikely(rc))
        security_task_free(task);
        
    return rc;
}
```

### File Descriptor Inheritance

#### File Table Duplication Strategy
```c
static int copy_files(unsigned long clone_flags, struct task_struct *tsk,
                     int no_files)
{
    struct files_struct *oldf, *newf;
    
    oldf = current->files;
    if (!oldf)
        return 0;
        
    if (clone_flags & CLONE_FILES) {
        atomic_inc(&oldf->count);
        return 0;
    }
    
    newf = dup_fd(oldf, NR_OPEN_DEFAULT, &error);
    if (!newf)
        return error;
        
    tsk->files = newf;
    return 0;
}
```

#### Detailed File Descriptor Copying
```c
struct files_struct *dup_fd(struct files_struct *oldf, unsigned int max_fds, int *errorp)
{
    struct files_struct *newf;
    struct file **old_fds, **new_fds;
    unsigned int open_files, i;
    struct fdtable *old_fdt, *new_fdt;
    
    *errorp = -ENOMEM;
    newf = kmem_cache_alloc(files_cachep, GFP_KERNEL);
    if (!newf)
        return NULL;
        
    atomic_set(&newf->count, 1);
    
    spin_lock_init(&newf->file_lock);
    newf->resize_in_progress = false;
    newf->next_fd = 0;
    new_fdt = &newf->fdtab;
    new_fdt->max_fds = NR_OPEN_DEFAULT;
    new_fdt->close_on_exec = newf->close_on_exec_init;
    new_fdt->open_fds = newf->open_fds_init;
    new_fdt->full_fds_bits = newf->full_fds_bits_init;
    new_fdt->fd = &newf->fd_array[0];
    
    spin_lock(&oldf->file_lock);
    old_fdt = files_fdtable(oldf);
    open_files = count_open_files(old_fdt);
    
    // Expand fd table if necessary
    while (unlikely(open_files > new_fdt->max_fds)) {
        spin_unlock(&oldf->file_lock);
        
        if (expand_files(newf, open_files-1)) {
            kmem_cache_free(files_cachep, newf);
            return NULL;
        }
        new_fdt = files_fdtable(newf);
        
        spin_lock(&oldf->file_lock);
        old_fdt = files_fdtable(oldf);
        open_files = count_open_files(old_fdt);
    }
    
    // Copy file descriptors
    old_fds = old_fdt->fd;
    new_fds = new_fdt->fd;
    
    for (i = open_files; i != 0; i--) {
        struct file *f = *old_fds++;
        if (f) {
            get_file(f);  // Increment reference count
        } else {
            // FD_CLOEXEC handling for closed descriptors
        }
        rcu_assign_pointer(*new_fds++, f);
    }
    
    // Copy close-on-exec bitmap
    memcpy(new_fdt->close_on_exec, old_fdt->close_on_exec,
           (new_fdt->max_fds + BITS_PER_LONG - 1) / BITS_PER_LONG * sizeof(long));
           
    memcpy(new_fdt->open_fds, old_fdt->open_fds,
           (new_fdt->max_fds + BITS_PER_LONG - 1) / BITS_PER_LONG * sizeof(long));
    
    spin_unlock(&oldf->file_lock);
    
    // Handle FD_CLOEXEC properly
    for (i = 0; i < new_fdt->max_fds; i++) {
        if (FD_ISSET(i, new_fdt->close_on_exec)) {
            // Handle close-on-exec semantics
        }
    }
    
    return newf;
}
```

### Filesystem Context Copying

#### Filesystem Information Duplication
```c
static int copy_fs(unsigned long clone_flags, struct task_struct *tsk)
{
    struct fs_struct *fs = current->fs;
    
    if (clone_flags & CLONE_FS) {
        fs->users++;
        return 0;
    }
    
    tsk->fs = copy_fs_struct(fs);
    if (!tsk->fs)
        return -ENOMEM;
        
    return 0;
}

struct fs_struct *copy_fs_struct(struct fs_struct *old)
{
    struct fs_struct *fs = kmem_cache_alloc(fs_cachep, GFP_KERNEL);
    
    if (fs) {
        fs->users = 1;
        fs->in_exec = 0;
        spin_lock_init(&fs->lock);
        seqcount_spinlock_init(&fs->seq, &fs->lock);
        fs->umask = old->umask;
        
        spin_lock(&old->lock);
        fs->root = old->root;
        path_get(&fs->root);
        fs->pwd = old->pwd;
        path_get(&fs->pwd);
        spin_unlock(&old->lock);
    }
    
    return fs;
}
```

### Namespace Inheritance and Creation

#### Namespace Copying Strategy
```c
static int copy_namespaces(unsigned long clone_flags, struct task_struct *tsk)
{
    struct nsproxy *old_ns = tsk->nsproxy;
    struct user_namespace *user_ns = task_cred_xxx(tsk, user_ns);
    struct nsproxy *new_ns;
    
    if (likely(!(clone_flags & (CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC |
                               CLONE_NEWPID | CLONE_NEWNET |
                               CLONE_NEWCGROUP | CLONE_NEWTIME)))) {
        if ((clone_flags & CLONE_NEWUSER) == 0)
            return 0;
    }
    
    if (!ns_capable(user_ns, CAP_SYS_ADMIN))
        return -EPERM;
        
    if ((clone_flags & CLONE_NEWUSER) && !unprivileged_userns_clone) {
        if (!capable(CAP_SYS_ADMIN))
            return -EPERM;
    }
    
    new_ns = create_new_namespaces(clone_flags, tsk, user_ns, tsk->fs);
    if (IS_ERR(new_ns))
        return PTR_ERR(new_ns);
        
    tsk->nsproxy = new_ns;
    return 0;
}
```

#### Individual Namespace Creation
```c
static struct nsproxy *create_new_namespaces(unsigned long flags,
    struct task_struct *tsk, struct user_namespace *user_ns,
    struct fs_struct *new_fs)
{
    struct nsproxy *new_nsp;
    int err;
    
    new_nsp = create_nsproxy();
    if (!new_nsp)
        return ERR_PTR(-ENOMEM);
        
    new_nsp->mnt_ns = copy_mnt_ns(flags, tsk->nsproxy->mnt_ns, user_ns, new_fs);
    if (IS_ERR(new_nsp->mnt_ns)) {
        err = PTR_ERR(new_nsp->mnt_ns);
        goto out_mount;
    }
    
    new_nsp->uts_ns = copy_utsname(flags, user_ns, tsk->nsproxy->uts_ns);
    if (IS_ERR(new_nsp->uts_ns)) {
        err = PTR_ERR(new_nsp->uts_ns);
        goto out_uts;
    }
    
    new_nsp->ipc_ns = copy_ipcs(flags, user_ns, tsk->nsproxy->ipc_ns);
    if (IS_ERR(new_nsp->ipc_ns)) {
        err = PTR_ERR(new_nsp->ipc_ns);
        goto out_ipc;
    }
    
    new_nsp->pid_ns_for_children =
        copy_pid_ns(flags, user_ns, tsk->nsproxy->pid_ns_for_children);
    if (IS_ERR(new_nsp->pid_ns_for_children)) {
        err = PTR_ERR(new_nsp->pid_ns_for_children);
        goto out_pid;
    }
    
    new_nsp->cgroup_ns = copy_cgroup_ns(flags, user_ns,
                                       tsk->nsproxy->cgroup_ns);
    if (IS_ERR(new_nsp->cgroup_ns)) {
        err = PTR_ERR(new_nsp->cgroup_ns);
        goto out_cgroup;
    }
    
    new_nsp->net_ns = copy_net_ns(flags, user_ns, tsk->nsproxy->net_ns);
    if (IS_ERR(new_nsp->net_ns)) {
        err = PTR_ERR(new_nsp->net_ns);
        goto out_net;
    }
    
    new_nsp->time_ns = copy_time_ns(flags, user_ns, tsk->nsproxy->time_ns);
    if (IS_ERR(new_nsp->time_ns)) {
        err = PTR_ERR(new_nsp->time_ns);
        goto out_time;
    }
    
    return new_nsp;
    
    // Cleanup on error...
}
```

## Agent 5: Scheduler Integration and Performance Optimizations

### Scheduler Initialization

#### `sched_fork()` - Scheduler Setup for New Task
```c
int sched_fork(unsigned long clone_flags, struct task_struct *p)
{
    __sched_fork(clone_flags, p);
    
    // Set task state to NEW
    p->__state = TASK_NEW;
    
    // Set up scheduling policy and priority
    p->prio = current->normal_prio;
    
    // Handle different scheduling classes
    if (unlikely(p->sched_reset_on_fork)) {
        if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
            p->policy = SCHED_NORMAL;
            p->static_prio = NICE_TO_PRIO(0);
            p->rt_priority = 0;
        } else if (PRIO_TO_NICE(p->static_prio) < 0)
            p->static_prio = NICE_TO_PRIO(0);
            
        p->prio = p->normal_prio = p->static_prio;
        set_load_weight(p, false);
        
        p->sched_reset_on_fork = 0;
    }
    
    if (dl_prio(p->prio))
        return -EAGAIN;
    else if (rt_prio(p->prio))
        p->sched_class = &rt_sched_class;
    else
        p->sched_class = &fair_sched_class;
        
    init_entity_runnable_average(&p->se);
    
    if (likely(sched_info_on()))
        memset(&p->sched_info, 0, sizeof(p->sched_info));
        
#if defined(CONFIG_SMP)
    p->on_cpu = 0;
#endif
    init_task_preempt_count(p);
#ifdef CONFIG_SMP
    plist_node_init(&p->pushable_tasks, MAX_PRIO);
    RB_CLEAR_NODE(&p->pushable_dl_tasks);
#endif
    return 0;
}
```

#### Basic Scheduler Initialization (`__sched_fork`)
```c
static void __sched_fork(unsigned long clone_flags, struct task_struct *p)
{
    p->on_rq = 0;
    
    p->se.on_rq = 0;
    p->se.exec_start = 0;
    p->se.sum_exec_runtime = 0;
    p->se.prev_sum_exec_runtime = 0;
    p->se.nr_migrations = 0;
    p->se.vruntime = 0;
    INIT_LIST_HEAD(&p->se.group_node);
    
#ifdef CONFIG_FAIR_GROUP_SCHED
    p->se.cfs_rq = NULL;
#endif
    
#ifdef CONFIG_SCHEDSTATS
    /* Even if schedstat is disabled, there should not be garbage */
    memset(&p->stats, 0, sizeof(p->stats));
#endif
    
    RB_CLEAR_NODE(&p->dl.rb_node);
    init_dl_task_timer(&p->dl);
    init_dl_inactive_task_timer(&p->dl);
    __dl_clear_params(p);
    
    INIT_LIST_HEAD(&p->rt.run_list);
    p->rt.timeout = 0;
    p->rt.time_slice = sched_rr_timeslice;
    p->rt.on_rq = 0;
    p->rt.on_list = 0;
    
#ifdef CONFIG_PREEMPT_NOTIFIERS
    INIT_HLIST_HEAD(&p->preempt_notifiers);
#endif
    
#ifdef CONFIG_COMPACTION
    p->capture_control = NULL;
#endif
    init_numa_balancing(clone_flags, p);
#ifdef CONFIG_SMP
    p->wake_entry.u64 = 0;
    p->migration_pending = NULL;
#endif
}
```

### CPU Assignment and Load Balancing

#### CPU Selection for New Task
```c
// In wake_up_new_task()
void wake_up_new_task(struct task_struct *p)
{
    struct rq_flags rf;
    struct rq *rq;
    
    raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
    p->__state = TASK_RUNNING;
    
#ifdef CONFIG_SMP
    /*
     * Fork balancing, do it here and not earlier because:
     *  - cpus_ptr can change in the fork path
     *  - any previously selected CPU might disappear through hotplug
     *  - because we're not yet on the tasklist, no one else can manipulate it
     */
    __set_task_cpu(p, select_task_rq(p, task_cpu(p), WF_FORK));
#endif
    
    rq = __task_rq_lock(p, &rf);
    update_rq_clock(rq);
    post_init_entity_util_avg(&p->se);
    
    activate_task(rq, p, ENQUEUE_NOCLOCK);
    trace_sched_wakeup_new(p);
    
    if (p->sched_class->task_woken) {
        /*
         * Nothing relies on rq->lock after this, so it's fine to
         * drop it.
         */
        rq_unpin_lock(rq, &rf);
        p->sched_class->task_woken(rq, p);
        rq_repin_lock(rq, &rf);
    }
    
    task_rq_unlock(rq, p, &rf);
}
```

#### NUMA-Aware CPU Selection
```c
static int select_task_rq(struct task_struct *p, int cpu, int flags)
{
    if (p->nr_cpus_allowed == 1)
        return cpu;
        
    if (dl_task(p))
        return select_task_rq_dl(p, cpu, flags);
    if (rt_task(p))
        return select_task_rq_rt(p, cpu, flags);
        
    return select_task_rq_fair(p, cpu, flags);
}

// Fair scheduler CPU selection with NUMA awareness
static int
select_task_rq_fair(struct task_struct *p, int prev_cpu, int wake_flags)
{
    int sync = (wake_flags & WF_SYNC) && !(current->flags & PF_EXITING);
    struct sched_domain *tmp, *sd = NULL;
    int cpu = smp_processor_id();
    int new_cpu = prev_cpu;
    int want_affine = 0;
    
    // Wake affinity optimization
    if (wake_flags & WF_TTWU) {
        record_wakee(p);
        
        if (sched_energy_enabled()) {
            new_cpu = find_energy_efficient_cpu(p, prev_cpu);
            if (new_cpu >= 0)
                return new_cpu;
            new_cpu = prev_cpu;
        }
        
        want_affine = !wake_wide(p) && cpumask_test_cpu(cpu, p->cpus_ptr);
    }
    
    rcu_read_lock();
    for_each_domain(cpu, tmp) {
        if (!(tmp->flags & SD_LOAD_BALANCE))
            break;
            
        // Want affine wake ups to stay on the same LLC
        if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&
            cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {
            if (cpu != prev_cpu)
                new_cpu = wake_affine(tmp, p, cpu, prev_cpu, sync);
                
            sd = NULL; // Prefer affine wakeup
            break;
        }
        
        if (tmp->flags & flags)
            sd = tmp;
        else if (!want_affine)
            break;
    }
    
    if (unlikely(sd)) {
        /* Slow path - find the best CPU in this domain */
        new_cpu = find_idlest_cpu(sd, p, cpu, prev_cpu, sd_flag);
    } else if (wake_flags & WF_TTWU) {
        /* Fast path for waking tasks */
        if (want_affine)
            new_cpu = cpu;
    }
    rcu_read_unlock();
    
    return new_cpu;
}
```

### Performance Optimizations

#### Stack Caching for Thread Creation
```c
static void thread_stack_free_rcu(struct rcu_head *rh)
{
    struct vm_struct *vm = container_of(rh, struct vm_struct, rcu_head);
    
    if (vm->flags & VM_KSTACK_GAP)
        vm->addr = (void *)((unsigned long)vm->addr + PAGE_SIZE);
        
    vfree(vm->addr);
}

static void free_thread_stack(struct task_struct *tsk)
{
    struct vm_struct *vm = task_stack_vm_area(tsk);
    
    if (vm) {
        int i;
        
        for (i = 0; i < THREAD_SIZE / PAGE_SIZE; i++)
            memcg_kmem_uncharge_page(vm->pages[i], 0);
            
        for (i = 0; i < NR_CACHED_STACKS; i++) {
            if (this_cpu_cmpxchg(cached_stacks[i], NULL,
                                tsk->stack_vm_area) != NULL)
                continue;
                
            return;
        }
        
        call_rcu(&vm->rcu_head, thread_stack_free_rcu);
        return;
    }
    
    __free_pages(virt_to_page(tsk->stack), THREAD_SIZE_ORDER);
}

static int alloc_thread_stack_node(struct task_struct *tsk, int node)
{
    struct vm_struct *vm;
    void *stack;
    int i;
    
    for (i = 0; i < NR_CACHED_STACKS; i++) {
        struct vm_struct *s;
        
        s = this_cpu_xchg(cached_stacks[i], NULL);
        if (!s)
            continue;
            
        // Check if cached stack is on the right node
        if (node == NUMA_NO_NODE || node == vm_node(s)) {
            tsk->stack_vm_area = s;
            tsk->stack = s->addr;
            return 0;
        }
        
        // Wrong node, put it back and continue
        this_cpu_cmpxchg(cached_stacks[i], NULL, s);
    }
    
    // Allocate new stack
    stack = __vmalloc_node_range(THREAD_SIZE, THREAD_ALIGN,
                                 VMALLOC_START, VMALLOC_END,
                                 THREADINFO_GFP,
                                 PAGE_KERNEL,
                                 0, node, __builtin_return_address(0));
    if (!stack)
        return -ENOMEM;
        
    vm = find_vm_area(stack);
    if (memcg_charge_kernel_stack(vm)) {
        vfree(stack);
        return -ENOMEM;
    }
    
    tsk->stack_vm_area = vm;
    tsk->stack = stack;
    return 0;
}
```

#### vfork() Optimization Implementation
```c
static int wait_for_vfork_done(struct task_struct *child,
                              struct completion *vfork)
{
    int killed;
    
    freezer_do_not_count();
    cgroup_enter_frozen();
    killed = wait_for_completion_state(vfork, state);
    cgroup_leave_frozen(false);
    freezer_count();
    
    if (killed) {
        task_lock(child);
        child->vfork_done = NULL;
        task_unlock(child);
    }
    
    put_task_struct(child);
    return killed;
}

static void complete_vfork_done(struct task_struct *tsk)
{
    struct completion *vfork;
    
    task_lock(tsk);
    vfork = tsk->vfork_done;
    if (likely(vfork)) {
        tsk->vfork_done = NULL;
        complete(vfork);
    }
    task_unlock(tsk);
}
```

### Memory and CPU Affinity Optimizations

#### NUMA Memory Allocation for Tasks
```c
static struct task_struct *alloc_task_struct_node(int node)
{
    return kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
}

static inline struct thread_info *alloc_thread_info_node(
    struct task_struct *tsk, int node)
{
    struct page *page = alloc_pages_node(node, THREADINFO_GFP,
                                        THREAD_SIZE_ORDER);
    
    if (likely(page)) {
        memcg_kmem_charge_page(page, GFP_KERNEL, THREAD_SIZE_ORDER);
        return page_address(page);
    }
    return NULL;
}
```

#### Load Balancing Integration
```c
// Called when a new task is created
void post_init_entity_util_avg(struct sched_entity *se)
{
    struct cfs_rq *cfs_rq = cfs_rq_of(se);
    struct sched_avg *sa = &se->avg;
    long cpu_scale = arch_scale_cpu_capacity(cpu_of(rq_of(cfs_rq)));
    long cap = (long)(cpu_scale - cfs_rq->avg.util_avg) / 2;
    
    if (cap > 0) {
        if (cfs_rq->avg.util_avg != 0) {
            sa->util_avg  = cfs_rq->avg.util_avg * se->load.weight;
            sa->util_avg /= (cfs_rq->avg.load_avg + 1);
            
            if (sa->util_avg > cap)
                sa->util_avg = cap;
        } else {
            sa->util_avg = cap;
        }
    }
    
    sa->runnable_avg = sa->util_avg;
}
```

## Comprehensive Error Handling and Resource Cleanup

### Structured Error Handling in `copy_process()`

The `copy_process()` function implements comprehensive error handling using a series of labeled cleanup sections:

```c
// Error cleanup labels in copy_process()
bad_fork_cleanup_threadgroup_lock:
    threadgroup_change_end(current);
bad_fork_cleanup_audit:
    audit_free(p);
bad_fork_cleanup_perf:
    perf_event_free_task(p);
bad_fork_cleanup_policy:
    lockdep_free_task(p);
#ifdef CONFIG_NUMA
    mpol_put(p->mempolicy);
#endif
bad_fork_cleanup_cgroup:
    cgroup_exit(p, 0);
bad_fork_cleanup_delayacct:
    delayacct_tsk_free(p);
bad_fork_cleanup_count:
    dec_rlimit_ucounts(task_ucounts(p), UCOUNT_RLIMIT_NPROC, 1);
    exit_creds(p);
bad_fork_free:
    put_task_struct(p);
fork_out:
    spin_lock_irq(&current->sighand->siglock);
    hlist_del_init(&delayed.node);
    spin_unlock_irq(&current->sighand->siglock);
    return ERR_PTR(retval);
```

### Performance Monitoring and Debugging

#### Fork Performance Counters
```c
// Global fork statistics
unsigned long total_forks;  // Total number of forks since boot

// Per-process fork accounting
struct task_struct {
    // ... other fields ...
    unsigned long nvcsw;     // Voluntary context switches
    unsigned long nivcsw;    // Involuntary context switches
    u64 utime, stime;       // User and system CPU time
    u64 gtime;              // Guest time
    unsigned long min_flt;   // Minor page faults
    unsigned long maj_flt;   // Major page faults
};
```

#### Memory Pressure Handling
```c
// Fork memory pressure detection
if (data_race(nr_threads >= max_threads)) {
    retval = -EAGAIN;
    goto bad_fork_cleanup_count;
}

// Memory allocation with fallback
static struct vm_struct *alloc_thread_stack_node(struct task_struct *tsk, int node)
{
    struct page *page = alloc_pages_node(node, THREADINFO_GFP, THREAD_SIZE_ORDER);
    
    if (!page) {
        // Try different allocation strategies
        page = alloc_pages_node(node, THREADINFO_GFP & ~__GFP_THISNODE, 
                               THREAD_SIZE_ORDER);
    }
    
    if (!page) {
        // Final fallback to any node
        page = alloc_pages(THREADINFO_GFP, THREAD_SIZE_ORDER);
    }
    
    return page ? virt_to_vmalloc(page_address(page)) : NULL;
}
```

## Integration with Modern Linux Features

### Container and Namespace Integration

#### Cgroup v2 Integration
```c
void cgroup_fork(struct task_struct *child)
{
    RCU_INIT_POINTER(child->cgroups, &init_css_set);
    INIT_LIST_HEAD(&child->cg_list);
}

int cgroup_can_fork(struct task_struct *child, struct kernel_clone_args *kargs)
{
    struct cgroup_subsys *ss;
    int i, j, ret;
    
    ret = cgroup_attach_permissions(kargs->cgrp, dst_cgrp, child,
                                   kargs->cset);
    if (ret)
        return ret;
        
    do_each_subsys_mask(ss, i, have_canfork_callback) {
        ret = ss->can_fork(child, kargs->cset);
        if (ret)
            goto out_revert;
    } while_each_subsys_mask();
    
    return 0;
    
out_revert:
    for_each_subsys(ss, j) {
        if (j >= i)
            break;
        if (ss->cancel_fork)
            ss->cancel_fork(child, kargs->cset);
    }
    
    return ret;
}
```

### Security Enhancements

#### KASLR and Stack Protection
```c
static void setup_new_task_stack_canary(struct task_struct *p)
{
#ifdef CONFIG_STACKPROTECTOR
    /*
     * Set up a new stack canary for the new task to prevent
     * stack-smashing attacks
     */
    unsigned long canary = get_random_canary();
    
    p->stack_canary = canary;
    
#ifdef CONFIG_X86_64
    /*
     * On x86_64, the stack canary is kept in a special
     * per-cpu variable
     */
    this_cpu_write(irq_stack_canary, canary);
#endif
#endif
}
```

#### Spectre/Meltdown Mitigations
```c
// Indirect branch prediction barrier
static void setup_task_isolation(struct task_struct *p)
{
#ifdef CONFIG_SPECULATION_MITIGATIONS
    // Set up task-specific speculation controls
    if (speculation_ctrl_enabled()) {
        p->thread.spec_ctrl = current->thread.spec_ctrl;
        
        // Apply indirect branch prediction barrier
        if (boot_cpu_has(X86_FEATURE_IBPB))
            native_wrmsrl(MSR_IA32_PRED_CMD, PRED_CMD_IBPB);
    }
#endif
}
```

## Summary

This implementation represents one of the most sophisticated process creation systems in any operating system, balancing performance, security, and flexibility while supporting modern containerization and threading requirements. The five-agent analysis reveals:

### Key Architectural Strengths

1. **Modular Design**: Clean separation between task creation, resource copying, and scheduler integration
2. **Scalability**: NUMA-aware allocation, CPU affinity management, and efficient resource sharing
3. **Security**: Comprehensive LSM integration, namespace isolation, and credential management
4. **Performance**: Copy-on-write memory management, stack caching, and optimized thread creation
5. **Robustness**: Extensive error handling, resource cleanup, and consistency guarantees

### Modern Linux Features

- **Container Support**: Full integration with cgroups, namespaces, and container runtimes
- **Security Hardening**: Stack protection, KASLR, speculation mitigations
- **Threading**: Efficient POSIX thread support with proper TLS handling
- **NUMA Optimization**: Memory and CPU placement for large systems
- **Real-time Support**: Integration with RT scheduling classes

### Performance Characteristics

- **Fork Latency**: Optimized for sub-millisecond process creation
- **Memory Efficiency**: COW semantics minimize memory overhead
- **CPU Overhead**: Intelligent CPU selection and load balancing
- **Scalability**: Linear scaling across thousands of processes/threads

The kernel/fork.c implementation demonstrates the evolution of Unix process creation from simple fork() to a sophisticated system supporting modern computing requirements including containerization, security isolation, and high-performance computing workloads.

---

**Analysis completed using specialized agent methodology:**
- **Agent 1**: Core fork/clone algorithms and task creation mechanisms
- **Agent 2**: Memory management during fork (COW and address space)
- **Agent 3**: Thread creation and thread group management
- **Agent 4**: Resource inheritance and security context copying
- **Agent 5**: Scheduler integration and performance optimizations

*Documentation generated through comprehensive kernel source analysis*