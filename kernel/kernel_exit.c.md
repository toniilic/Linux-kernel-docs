# Linux Kernel Process Exit Implementation

## Table of Contents
1. [Overview](#overview)
2. [Agent 1: Core Exit Process Flow and Task Cleanup](#agent-1-core-exit-process-flow-and-task-cleanup)
3. [Agent 2: Memory Management Cleanup and Resource Deallocation](#agent-2-memory-management-cleanup-and-resource-deallocation)
4. [Agent 3: Parent-Child Relationships and Wait Handling](#agent-3-parent-child-relationships-and-wait-handling)
5. [Agent 4: Thread Group Exit and Signal Handling](#agent-4-thread-group-exit-and-signal-handling)
6. [Agent 5: Namespace Cleanup and Security Context Teardown](#agent-5-namespace-cleanup-and-security-context-teardown)
7. [Integration with Scheduler and Memory Management](#integration-with-scheduler-and-memory-management)
8. [Error Handling and Edge Cases](#error-handling-and-edge-cases)
9. [System Call Interface](#system-call-interface)
10. [Data Structures Reference](#data-structures-reference)

## Overview

The Linux kernel's process exit implementation in `kernel/exit.c` is a critical subsystem responsible for properly terminating processes and cleaning up their resources. This comprehensive analysis covers the complex orchestration of cleanup operations, parent-child relationship management, and system state consistency during process termination.

### Key Entry Points
- `do_exit(long code)` - Main exit function for normal termination
- `make_task_dead(int signr)` - Catastrophic failure termination
- `do_group_exit(int exit_code)` - Terminate entire thread group
- System calls: `exit()`, `exit_group()`, `wait4()`, `waitid()`, `waitpid()`

### Core Data Structures
- `struct task_struct` - Process control block
- `struct signal_struct` - Signal and thread group management
- `struct wait_opts` - Wait operation parameters
- `struct release_task_post` - Post-release cleanup coordination

---

## Agent 1: Core Exit Process Flow and Task Cleanup

### Main Exit Flow: `do_exit()`

The `do_exit()` function orchestrates the complete process termination sequence:

```c
void __noreturn do_exit(long code)
{
    struct task_struct *tsk = current;
    int group_dead;
    
    // 1. Early validation and setup
    WARN_ON(irqs_disabled());
    WARN_ON(tsk->plug);
    
    // 2. Coverage and memory sanitizer cleanup
    kcov_task_exit(tsk);
    kmsan_task_exit(tsk);
    
    // 3. Thread group synchronization
    synchronize_group_exit(tsk, code);
    
    // 4. Ptrace and user event notification
    ptrace_event(PTRACE_EVENT_EXIT, code);
    user_events_exit(tsk);
    
    // 5. File and signal cleanup
    io_uring_files_cancel();
    exit_signals(tsk);  // Sets PF_EXITING
    
    // 6. Security and accounting
    seccomp_filter_release(tsk);
    acct_update_integrals(tsk);
    
    // 7. Resource cleanup sequence
    exit_mm();          // Memory management
    exit_sem(tsk);      // System V semaphores
    exit_shm(tsk);      // Shared memory
    exit_files(tsk);    // File descriptors
    exit_fs(tsk);       // Filesystem context
    
    // 8. Namespace and control group cleanup
    exit_task_namespaces(tsk);
    exit_task_work(tsk);
    exit_thread(tsk);
    sched_autogroup_exit_task(tsk);
    cgroup_exit(tsk);
    
    // 9. Final state transition and notification
    exit_notify(tsk, group_dead);
    
    // 10. Final cleanup and scheduler handoff
    exit_rcu();
    exit_tasks_rcu_finish();
    lockdep_free_task(tsk);
    do_task_dead();     // Never returns
}
```

### Task State Transitions

Process exit involves several state transitions managed through `task->exit_state`:

1. **TASK_RUNNING** → **PF_EXITING flag set** (via `exit_signals()`)
2. **Active** → **EXIT_ZOMBIE** (via `exit_notify()`)
3. **EXIT_ZOMBIE** → **EXIT_DEAD** (via parent's `wait()` or auto-reap)
4. **EXIT_DEAD** → **Released** (via `release_task()`)

```c
// State transition in exit_notify()
tsk->exit_state = EXIT_ZOMBIE;

// Later transition in wait_task_zombie()
state = (ptrace_reparented(p) && thread_group_leader(p)) ?
        EXIT_TRACE : EXIT_DEAD;
if (cmpxchg(&p->exit_state, EXIT_ZOMBIE, state) != EXIT_ZOMBIE)
    return 0;
```

### Exit Code Handling

The kernel preserves exit codes through multiple layers:

```c
// In do_exit()
tsk->exit_code = code;

// For thread groups, group exit code takes precedence
if (group_dead) {
    if (leader->signal->flags & SIGNAL_GROUP_EXIT)
        leader->exit_code = leader->signal->group_exit_code;
}
```

### Task Cleanup Sequence

The cleanup follows a careful ordering to prevent races and ensure consistency:

1. **Early preparation**: Coverage tools, ptrace notification
2. **Resource isolation**: Signal blocking, file cancellation
3. **Memory cleanup**: MM structures, page tables
4. **IPC cleanup**: Semaphores, shared memory
5. **File system cleanup**: File descriptors, filesystem context
6. **Namespace cleanup**: Network, mount, PID namespaces
7. **Final notification**: Parent notification, zombie creation

### Critical Sections and Locking

The exit path uses several locking mechanisms:

```c
// Thread group synchronization
synchronize_group_exit(tsk, code);

// Tasklist protection during parent notification
write_lock_irq(&tasklist_lock);
exit_notify(tsk, group_dead);
write_unlock_irq(&tasklist_lock);

// Signal lock for group coordination
spin_lock_irq(&sighand->siglock);
// ... group exit coordination
spin_unlock_irq(&sighand->siglock);
```

---

## Agent 2: Memory Management Cleanup and Resource Deallocation

### Memory Map Cleanup: `exit_mm()`

The `exit_mm()` function performs comprehensive memory management cleanup:

```c
static void exit_mm(void)
{
    struct mm_struct *mm = current->mm;
    
    // Release MM-related resources for current task
    exit_mm_release(current, mm);
    if (!mm)
        return;
        
    // Acquire lazy TLB reference for cleanup
    mmap_read_lock(mm);
    mmgrab_lazy_tlb(mm);
    BUG_ON(mm != current->active_mm);
    
    // Memory barrier for membarrier() synchronization
    task_lock(current);
    smp_mb__after_spinlock();
    local_irq_disable();
    
    // Clear current task's MM pointer
    current->mm = NULL;
    membarrier_update_current_mm(NULL);
    enter_lazy_tlb(mm, current);
    local_irq_enable();
    task_unlock(current);
    
    // Release read lock and update ownership
    mmap_read_unlock(mm);
    mm_update_next_owner(mm);
    
    // Final MM reference drop (may trigger cleanup)
    mmput(mm);
    
    // Handle OOM victim cleanup
    if (test_thread_flag(TIF_MEMDIE))
        exit_oom_victim();
}
```

### MM Ownership Transfer

When a process exits, the kernel must handle MM ownership transfer for shared memory spaces:

```c
void mm_update_next_owner(struct mm_struct *mm)
{
    struct task_struct *g, *p = current;
    
    // Only process if current task owns the MM
    if (mm->owner != p)
        return;
        
    // Handle last user case
    if (atomic_read(&mm->mm_users) <= 1) {
        WRITE_ONCE(mm->owner, NULL);
        return;
    }
    
    read_lock(&tasklist_lock);
    
    // Search priority order:
    // 1. Children of current process
    list_for_each_entry(g, &p->children, sibling) {
        if (try_to_set_owner(g, mm))
            goto ret;
    }
    
    // 2. Siblings of current process
    list_for_each_entry(g, &p->real_parent->children, sibling) {
        if (try_to_set_owner(g, mm))
            goto ret;
    }
    
    // 3. All other processes (last resort)
    for_each_process(g) {
        if (atomic_read(&mm->mm_users) <= 1)
            break;
        if (g->flags & PF_KTHREAD)
            continue;
        if (try_to_set_owner(g, mm))
            goto ret;
    }
    
    // No suitable owner found
    WRITE_ONCE(mm->owner, NULL);
ret:
    read_unlock(&tasklist_lock);
}
```

### Page Table and VMA Cleanup

Memory cleanup involves several layers handled by the MM subsystem:

1. **VMA teardown**: Virtual memory areas are unmapped and freed
2. **Page table cleanup**: Page tables are walked and freed
3. **Physical page release**: Physical pages are returned to allocator
4. **MM structure cleanup**: Final mm_struct cleanup

The actual VMA and page table cleanup occurs in `exit_mmap()` (called via `mmput()`):

```c
// In mm/mmap.c
void exit_mmap(struct mm_struct *mm)
{
    struct vm_area_struct *vma;
    
    // Flush TLB and release all VMAs
    mmap_write_lock(mm);
    arch_exit_mmap(mm);
    
    // Free all VMAs
    while ((vma = remove_vma(vma))) {
        // VMA cleanup handled by remove_vma()
    }
    mmap_write_unlock(mm);
}
```

### File Descriptor Cleanup

File descriptor cleanup is handled by `exit_files()`:

```c
// In fs/file.c
void exit_files(struct task_struct *tsk)
{
    struct files_struct *files = tsk->files;
    
    if (files) {
        task_lock(tsk);
        tsk->files = NULL;
        task_unlock(tsk);
        put_files_struct(files);
    }
}
```

### Filesystem Context Cleanup

Filesystem context cleanup via `exit_fs()`:

```c
// In fs/fs_struct.c  
void exit_fs(struct task_struct *tsk)
{
    struct fs_struct *fs = tsk->fs;
    
    if (fs) {
        int kill;
        task_lock(tsk);
        tsk->fs = NULL;
        kill = !--fs->users;
        task_unlock(tsk);
        if (kill)
            free_fs_struct(fs);
    }
}
```

### Resource Accounting and Limits

The exit path updates resource usage statistics:

```c
// Update integral accounting
acct_update_integrals(tsk);

// Collect accounting information for group
acct_collect(code, group_dead);

// Update max RSS for thread group
if (group_dead && tsk->mm)
    setmax_mm_hiwater_rss(&tsk->signal->maxrss, tsk->mm);
```

### Memory Cleanup Edge Cases

1. **Shared MM structures**: Multiple tasks sharing MM (threads)
2. **Lazy TLB**: Kernel threads borrowing MM for efficiency
3. **Memory cgroups**: Accounting and limit enforcement
4. **OOM conditions**: Special handling for OOM victim cleanup
5. **Coredump synchronization**: Preventing MM cleanup during coredump

---

## Agent 3: Parent-Child Relationships and Wait Handling

### Parent Notification: `exit_notify()`

The `exit_notify()` function manages the complex process of notifying parents and handling child reaping:

```c
static void exit_notify(struct task_struct *tsk, int group_dead)
{
    bool autoreap;
    struct task_struct *p, *n;
    LIST_HEAD(dead);
    
    write_lock_irq(&tasklist_lock);
    
    // Handle child reparenting
    forget_original_parent(tsk, &dead);
    
    // Check for orphaned process groups
    if (group_dead)
        kill_orphaned_pgrp(tsk->group_leader, NULL);
    
    // Transition to zombie state
    tsk->exit_state = EXIT_ZOMBIE;
    
    // Determine if auto-reaping should occur
    if (unlikely(tsk->ptrace)) {
        int sig = thread_group_leader(tsk) &&
                  thread_group_empty(tsk) &&
                  !ptrace_reparented(tsk) ?
                  tsk->exit_signal : SIGCHLD;
        autoreap = do_notify_parent(tsk, sig);
    } else if (thread_group_leader(tsk)) {
        autoreap = thread_group_empty(tsk) &&
                   do_notify_parent(tsk, tsk->exit_signal);
    } else {
        autoreap = true;
        do_notify_pidfd(tsk);  // Notify pidfd waiters
    }
    
    // Handle auto-reaping
    if (autoreap) {
        tsk->exit_state = EXIT_DEAD;
        list_add(&tsk->ptrace_entry, &dead);
    }
    
    // Wake up group exec task if needed
    if (unlikely(tsk->signal->notify_count < 0))
        wake_up_process(tsk->signal->group_exec_task);
        
    write_unlock_irq(&tasklist_lock);
    
    // Clean up auto-reaped tasks
    list_for_each_entry_safe(p, n, &dead, ptrace_entry) {
        list_del_init(&p->ptrace_entry);
        release_task(p);
    }
}
```

### Child Reparenting: `forget_original_parent()`

When a process exits, its children must be reparented:

```c
static void forget_original_parent(struct task_struct *father,
                                   struct list_head *dead)
{
    struct task_struct *p, *t, *reaper;
    
    // Handle ptraced children first
    if (unlikely(!list_empty(&father->ptraced)))
        exit_ptrace(father, dead);
    
    // Find appropriate reaper
    reaper = find_child_reaper(father, dead);
    if (list_empty(&father->children))
        return;
    
    reaper = find_new_reaper(father, reaper);
    
    // Reparent all children
    list_for_each_entry(p, &father->children, sibling) {
        for_each_thread(p, t) {
            // Update parent pointers
            RCU_INIT_POINTER(t->real_parent, reaper);
            if (likely(!t->ptrace))
                t->parent = t->real_parent;
                
            // Send parent death signal if configured
            if (t->pdeath_signal)
                group_send_sig_info(t->pdeath_signal,
                                   SEND_SIG_NOINFO, t,
                                   PIDTYPE_TGID);
        }
        
        // Handle thread group leader reparenting
        if (!same_thread_group(reaper, father))
            reparent_leader(father, p, dead);
    }
    
    // Transfer children list to reaper
    list_splice_tail_init(&father->children, &reaper->children);
}
```

### Reaper Selection Algorithm

The kernel uses a sophisticated algorithm to find appropriate reapers:

```c
static struct task_struct *find_new_reaper(struct task_struct *father,
                                          struct task_struct *child_reaper)
{
    struct task_struct *thread, *reaper;
    
    // 1. Try to find another thread in the same thread group
    thread = find_alive_thread(father);
    if (thread)
        return thread;
    
    // 2. Look for child subreaper in ancestry
    if (father->signal->has_child_subreaper) {
        unsigned int ns_level = task_pid(father)->level;
        
        for (reaper = father->real_parent;
             task_pid(reaper)->level == ns_level;
             reaper = reaper->real_parent) {
             
            if (reaper == &init_task)
                break;
            if (!reaper->signal->is_child_subreaper)
                continue;
                
            thread = find_alive_thread(reaper);
            if (thread)
                return thread;
        }
    }
    
    // 3. Fall back to namespace init (PID 1)
    return child_reaper;
}
```

### Wait System Call Implementation

The wait family of system calls provides the interface for parent processes to collect child exit status:

#### Core Wait Logic: `do_wait()`

```c
static long do_wait(struct wait_opts *wo)
{
    int retval;
    
    trace_sched_process_wait(wo->wo_pid);
    
    // Set up wait queue for child exit notifications
    init_waitqueue_func_entry(&wo->child_wait, child_wait_callback);
    wo->child_wait.private = current;
    add_wait_queue(&current->signal->wait_chldexit, &wo->child_wait);
    
    do {
        set_current_state(TASK_INTERRUPTIBLE);
        retval = __do_wait(wo);
        if (retval != -ERESTARTSYS)
            break;
        if (signal_pending(current))
            break;
        schedule();  // Sleep until child state change
    } while (1);
    
    __set_current_state(TASK_RUNNING);
    remove_wait_queue(&current->signal->wait_chldexit, &wo->child_wait);
    return retval;
}
```

#### Zombie Process Handling: `wait_task_zombie()`

```c
static int wait_task_zombie(struct wait_opts *wo, struct task_struct *p)
{
    int state, status;
    pid_t pid = task_pid_vnr(p);
    uid_t uid = from_kuid_munged(current_user_ns(), task_uid(p));
    
    if (!likely(wo->wo_flags & WEXITED))
        return 0;
    
    // Handle WNOWAIT case (peek at status without reaping)
    if (unlikely(wo->wo_flags & WNOWAIT)) {
        status = (p->signal->flags & SIGNAL_GROUP_EXIT) ?
                 p->signal->group_exit_code : p->exit_code;
        // Return status without reaping
        get_task_struct(p);
        read_unlock(&tasklist_lock);
        // ... handle rusage and return
        return pid;
    }
    
    // Atomically transition EXIT_ZOMBIE -> EXIT_DEAD/EXIT_TRACE
    state = (ptrace_reparented(p) && thread_group_leader(p)) ?
            EXIT_TRACE : EXIT_DEAD;
    if (cmpxchg(&p->exit_state, EXIT_ZOMBIE, state) != EXIT_ZOMBIE)
        return 0;
    
    read_unlock(&tasklist_lock);
    
    // Collect resource usage statistics
    if (state == EXIT_DEAD && thread_group_leader(p)) {
        // Accumulate thread group statistics in parent
        struct signal_struct *sig = p->signal;
        struct signal_struct *psig = current->signal;
        
        thread_group_cputime_adjusted(p, &tgutime, &tgstime);
        write_seqlock_irq(&psig->stats_lock);
        psig->cutime += tgutime + sig->cutime;
        psig->cstime += tgstime + sig->cstime;
        // ... more stat accumulation
        write_sequnlock_irq(&psig->stats_lock);
    }
    
    // Handle ptrace case
    if (state == EXIT_TRACE) {
        write_lock_irq(&tasklist_lock);
        ptrace_unlink(p);
        
        state = EXIT_ZOMBIE;
        if (do_notify_parent(p, p->exit_signal))
            state = EXIT_DEAD;
        p->exit_state = state;
        write_unlock_irq(&tasklist_lock);
    }
    
    // Final cleanup
    if (state == EXIT_DEAD)
        release_task(p);
    
    return pid;
}
```

### Wait Options and Filtering

The wait subsystem supports various filtering and behavior options:

```c
struct wait_opts {
    enum pid_type       wo_type;     // PIDTYPE_PID, PIDTYPE_PGID, etc.
    int                 wo_flags;    // WNOHANG, WUNTRACED, WCONTINUED, etc.
    struct pid         *wo_pid;      // Target PID or NULL for any
    struct waitid_info *wo_info;     // Extended status information
    int                 wo_stat;     // Traditional wait status
    struct rusage      *wo_rusage;   // Resource usage information
    wait_queue_entry_t  child_wait;  // Wait queue entry
    int                 notask_error; // Error when no eligible children
};
```

### Orphan Process Group Handling

The kernel tracks and manages orphaned process groups:

```c
static void kill_orphaned_pgrp(struct task_struct *tsk, struct task_struct *parent)
{
    struct pid *pgrp = task_pgrp(tsk);
    struct task_struct *ignored_task = tsk;
    
    if (!parent) {
        parent = tsk->real_parent;
    } else {
        ignored_task = NULL;
    }
    
    // Check if process group becomes orphaned
    if (task_pgrp(parent) != pgrp &&
        task_session(parent) == task_session(tsk) &&
        will_become_orphaned_pgrp(pgrp, ignored_task) &&
        has_stopped_jobs(pgrp)) {
        
        // Send SIGHUP and SIGCONT to orphaned group with stopped jobs
        __kill_pgrp_info(SIGHUP, SEND_SIG_PRIV, pgrp);
        __kill_pgrp_info(SIGCONT, SEND_SIG_PRIV, pgrp);
    }
}
```

---

## Agent 4: Thread Group Exit and Signal Handling

### Thread Group Coordination

Thread group exit requires careful coordination to ensure all threads terminate properly and resources are cleaned up correctly.

#### Group Exit Synchronization: `synchronize_group_exit()`

```c
static void synchronize_group_exit(struct task_struct *tsk, long code)
{
    struct sighand_struct *sighand = tsk->sighand;
    struct signal_struct *signal = tsk->signal;
    struct core_state *core_state;
    
    spin_lock_irq(&sighand->siglock);
    
    // Decrement quick thread count
    signal->quick_threads--;
    
    // Set group exit flag if this is the last quick thread
    if ((signal->quick_threads == 0) &&
        !(signal->flags & SIGNAL_GROUP_EXIT)) {
        signal->flags = SIGNAL_GROUP_EXIT;
        signal->group_exit_code = code;
        signal->group_stop_count = 0;
    }
    
    // Handle coredump synchronization
    tsk->flags |= PF_POSTCOREDUMP;
    core_state = signal->core_state;
    spin_unlock_irq(&sighand->siglock);
    
    // Wait for coredump completion if in progress
    if (unlikely(core_state))
        coredump_task_exit(tsk, core_state);
}
```

#### Group Exit Initiation: `do_group_exit()`

```c
void __noreturn do_group_exit(int exit_code)
{
    struct signal_struct *sig = current->signal;
    
    // Check if group exit already in progress
    if (sig->flags & SIGNAL_GROUP_EXIT)
        exit_code = sig->group_exit_code;
    else if (sig->group_exec_task)
        exit_code = 0;  // exec in progress
    else {
        struct sighand_struct *const sighand = current->sighand;
        
        spin_lock_irq(&sighand->siglock);
        
        // Double-check after acquiring lock
        if (sig->flags & SIGNAL_GROUP_EXIT)
            exit_code = sig->group_exit_code;
        else if (sig->group_exec_task)
            exit_code = 0;
        else {
            // Initiate group exit
            sig->group_exit_code = exit_code;
            sig->flags = SIGNAL_GROUP_EXIT;
            zap_other_threads(current);  // Kill other threads
        }
        spin_unlock_irq(&sighand->siglock);
    }
    
    do_exit(exit_code);
}
```

### Signal Cleanup During Exit

The signal subsystem requires careful cleanup during process exit:

#### Signal Structure Cleanup: `__exit_signal()`

```c
static void __exit_signal(struct release_task_post *post, struct task_struct *tsk)
{
    struct signal_struct *sig = tsk->signal;
    bool group_dead = thread_group_leader(tsk);
    struct sighand_struct *sighand;
    struct tty_struct *tty;
    u64 utime, stime;
    
    sighand = rcu_dereference_check(tsk->sighand,
                                   lockdep_tasklist_lock_is_held());
    spin_lock(&sighand->siglock);
    
    // Clean up POSIX timers
    #ifdef CONFIG_POSIX_TIMERS
    posix_cpu_timers_exit(tsk);
    if (group_dead)
        posix_cpu_timers_exit_group(tsk);
    #endif
    
    // Handle TTY cleanup for group leader
    if (group_dead) {
        tty = sig->tty;
        sig->tty = NULL;
    } else {
        // Wake up waiting exec task
        if (sig->notify_count > 0 && !--sig->notify_count)
            wake_up_process(sig->group_exec_task);
        
        // Update current target for signal delivery
        if (tsk == sig->curr_target)
            sig->curr_target = next_thread(tsk);
    }
    
    // Accumulate thread statistics
    task_cputime(tsk, &utime, &stime);
    write_seqlock(&sig->stats_lock);
    sig->utime += utime;
    sig->stime += stime;
    sig->gtime += task_gtime(tsk);
    sig->min_flt += tsk->min_flt;
    sig->maj_flt += tsk->maj_flt;
    sig->nvcsw += tsk->nvcsw;
    sig->nivcsw += tsk->nivcsw;
    sig->inblock += task_io_get_inblock(tsk);
    sig->oublock += task_io_get_oublock(tsk);
    task_io_accounting_add(&sig->ioac, &tsk->ioac);
    sig->sum_sched_runtime += tsk->se.sum_exec_runtime;
    sig->nr_threads--;
    
    // Unhash process from PID tables
    __unhash_process(post, tsk, group_dead);
    write_sequnlock(&sig->stats_lock);
    
    // Clean up signal handler
    tsk->sighand = NULL;
    spin_unlock(&sighand->siglock);
    
    __cleanup_sighand(sighand);
    if (group_dead)
        tty_kref_put(tty);
}
```

#### Process Unhashing: `__unhash_process()`

```c
static void __unhash_process(struct release_task_post *post, struct task_struct *p,
                            bool group_dead)
{
    struct pid *pid = task_pid(p);
    
    nr_threads--;  // Global thread count
    
    // Detach from PID namespace
    detach_pid(post->pids, p, PIDTYPE_PID);
    wake_up_all(&pid->wait_pidfd);  // Wake pidfd waiters
    
    if (group_dead) {
        // Remove from all PID types for group leader
        detach_pid(post->pids, p, PIDTYPE_TGID);
        detach_pid(post->pids, p, PIDTYPE_PGID);
        detach_pid(post->pids, p, PIDTYPE_SID);
        
        // Remove from global task lists
        list_del_rcu(&p->tasks);
        list_del_init(&p->sibling);
        __this_cpu_dec(process_counts);
    }
    
    // Remove from thread group list
    list_del_rcu(&p->thread_node);
}
```

### Coredump Integration

The exit path must coordinate with ongoing coredumps:

```c
static void coredump_task_exit(struct task_struct *tsk,
                              struct core_state *core_state)
{
    struct core_thread self;
    
    self.task = tsk;
    if (self.task->flags & PF_SIGNALED)
        self.next = xchg(&core_state->dumper.next, &self);
    else
        self.task = NULL;
    
    // Signal completion if this is the last thread
    if (atomic_dec_and_test(&core_state->nr_threads))
        complete(&core_state->startup);
    
    // Wait for coredump completion
    for (;;) {
        set_current_state(TASK_IDLE|TASK_FREEZABLE);
        if (!self.task)  // Released by coredump_finish()
            break;
        schedule();
    }
    __set_current_state(TASK_RUNNING);
}
```

### Signal Delivery During Exit

The kernel ensures proper signal handling during the exit process:

1. **Early signal blocking**: `exit_signals()` sets PF_EXITING flag
2. **Signal queue cleanup**: Pending signals are flushed
3. **Handler cleanup**: Signal handlers are cleaned up with sighand_struct
4. **Group signal coordination**: Group-wide signals are properly handled

#### Signal Exit Sequence

```c
// In kernel/signal.c
void exit_signals(struct task_struct *tsk)
{
    int group_stop = 0;
    sigset_t unblocked;
    
    // Disable signal delivery to this task
    tsk->flags |= PF_EXITING;
    
    // Handle group stop synchronization
    if (unlikely(tsk->signal->flags & SIGNAL_GROUP_EXIT)) {
        group_stop = !!(tsk->signal->flags & SIGNAL_GROUP_STOP);
        if (group_stop)
            tsk->jobctl |= JOBCTL_STOP_PENDING;
    }
    
    // ... additional signal exit handling
}
```

### Thread Group Leader Handling

Special consideration is given to thread group leaders:

1. **Resource accumulation**: Leader accumulates statistics from all threads
2. **PID namespace management**: Leader holds primary PID references
3. **Parent notification**: Only leaders notify parents (usually)
4. **Signal delivery**: Group signals target the leader
5. **Exit code preservation**: Leader's exit code represents the group

---

## Agent 5: Namespace Cleanup and Security Context Teardown

### Namespace Reference Management

Process exit requires careful cleanup of namespace references to prevent resource leaks and maintain system integrity.

#### Task Namespace Cleanup: `exit_task_namespaces()`

```c
// In kernel/nsproxy.c
void exit_task_namespaces(struct task_struct *p)
{
    switch_task_namespaces(p, NULL);
}

void switch_task_namespaces(struct task_struct *p, struct nsproxy *new)
{
    struct nsproxy *ns;
    
    might_sleep();
    
    task_lock(p);
    ns = p->nsproxy;
    p->nsproxy = new;
    task_unlock(p);
    
    if (ns && atomic_dec_and_test(&ns->count))
        free_nsproxy(ns);
}
```

#### Namespace Teardown Sequence

The kernel maintains references to multiple namespace types:

```c
struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;      // Hostname/domain
    struct ipc_namespace *ipc_ns;      // System V IPC
    struct mnt_namespace *mnt_ns;      // Mount points
    struct pid_namespace *pid_ns;      // Process IDs
    struct net *net_ns;                // Network stack
    struct time_namespace *time_ns;    // Time offsets
    struct cgroup_namespace *cgroup_ns; // Control groups
};
```

Each namespace type has specific cleanup requirements:

1. **UTS Namespace**: Hostname and domain name isolation
2. **IPC Namespace**: System V semaphores, message queues, shared memory
3. **Mount Namespace**: Filesystem mount topology
4. **PID Namespace**: Process ID space and reaping
5. **Network Namespace**: Network devices, routing, netfilter
6. **Time Namespace**: Clock offsets for processes
7. **Cgroup Namespace**: Control group view isolation

### PID Namespace Exit Handling

PID namespace exit is particularly complex due to the reaping responsibilities:

```c
// In kernel/pid_namespace.c
void zap_pid_ns_processes(struct pid_namespace *pid_ns)
{
    int nr;
    int rc;
    struct task_struct *task, *me = current;
    int init_pids = thread_group_leader(me) ? 1 : 2;
    struct pid *pid;
    
    // Ignore SIGCHLD to auto-reap children
    me->sighand->action[SIGCHLD - 1].sa.sa_handler = SIG_IGN;
    
    // Kill all processes in this namespace
    for (nr = 0; nr < PIDMAP_ENTRIES; nr++) {
        if (nr == task_active_pid_ns(me)->last_pid)
            continue;
            
        pid = find_pid_ns(nr, pid_ns);
        if (pid == NULL)
            continue;
            
        task = pid_task(pid, PIDTYPE_PID);
        if (task && !__fatal_signal_pending(task))
            send_sig_info(SIGKILL, SEND_SIG_PRIV, task);
    }
    
    // Wait for all processes to exit
    do {
        clear_thread_flag(TIF_SIGPENDING);
        rc = sys_wait4(-1, NULL, __WALL, NULL);
    } while (rc != -ECHILD);
    
    // Final cleanup
    acct_exit_ns(pid_ns);
}
```

### Credential and Security Context Cleanup

Process exit involves cleanup of security-related structures:

#### Credential Management

```c
// Credentials are cleaned up via put_cred() calls throughout exit
void put_cred(const struct cred *_cred)
{
    struct cred *cred = (struct cred *) _cred;
    
    if (atomic_dec_and_test(&cred->usage))
        __put_cred(cred);
}

void __put_cred(struct cred *cred)
{
    // Clean up various credential components
    put_uid(cred->uid);
    put_gid(cred->gid);
    put_uid(cred->suid);
    put_gid(cred->sgid);
    put_uid(cred->euid);
    put_gid(cred->egid);
    put_uid(cred->fsuid);
    put_gid(cred->fsgid);
    put_group_info(cred->group_info);
    put_user_ns(cred->user_ns);
    
    // Security module cleanup
    security_cred_free(cred);
    key_put(cred->session_keyring);
    key_put(cred->process_keyring);
    key_put(cred->thread_keyring);
    
    kfree(cred);
}
```

#### Capability Cleanup

Capabilities are cleaned up as part of credential cleanup:

```c
// Capability sets are freed with the credential structure
struct cred {
    // ... other fields
    kernel_cap_t    cap_inheritable; // Inheritable capabilities
    kernel_cap_t    cap_permitted;   // Permitted capabilities
    kernel_cap_t    cap_effective;   // Effective capabilities
    kernel_cap_t    cap_bset;        // Capability bounding set
    kernel_cap_t    cap_ambient;     // Ambient capabilities
    // ... other fields
};
```

### Security Module Integration

The exit path integrates with Linux Security Modules (LSMs):

```c
// Security hooks are called throughout the exit process
void security_task_free(struct task_struct *task)
{
    call_void_hook(task_free, task);
}

// LSM-specific cleanup (example: SELinux)
static void selinux_task_free(struct task_struct *task)
{
    struct task_security_struct *tsec = task->security;
    
    if (!tsec)
        return;
        
    task->security = NULL;
    kfree(tsec);
}
```

### Audit Subsystem Cleanup

The audit subsystem maintains per-task context that requires cleanup:

```c
// In kernel/audit.c
void audit_free(struct task_struct *tsk)
{
    struct audit_context *context;
    
    context = audit_get_context(tsk, 0, 0);
    if (!context)
        return;
        
    audit_log_exit(context, tsk);
    audit_free_context(context);
}
```

### Key Management Cleanup

Process keyrings and key references are cleaned up:

```c
// In security/keys/process_keys.c
void key_put(struct key *key)
{
    if (key) {
        if (atomic_dec_and_test(&key->usage))
            schedule_work(&key_gc_work);
    }
}

void exit_keys(struct task_struct *tsk)
{
    key_put(tsk->thread_keyring);
    key_put(tsk->request_key_auth);
    tsk->thread_keyring = NULL;
    tsk->request_key_auth = NULL;
}
```

### User Namespace Reference Handling

User namespace references require careful management:

```c
void put_user_ns(struct user_namespace *ns)
{
    if (ns && atomic_dec_and_test(&ns->count))
        schedule_work(&cleanup_user_ns_work);
}

static void cleanup_user_ns(struct work_struct *work)
{
    struct user_namespace *ns;
    // ... cleanup user namespace resources
}
```

### Control Group Integration

Control group cleanup during process exit:

```c
// In kernel/cgroup/cgroup.c
void cgroup_exit(struct task_struct *tsk)
{
    struct cgroup_subsys *ss;
    struct css_set *cset;
    int i;
    
    // Exit hooks for each cgroup subsystem
    do_each_subsys_mask(ss, i, have_exit_callback) {
        ss->exit(tsk);
    } while_each_subsys_mask();
    
    // Release CSS set reference
    cset = task_css_set(tsk);
    if (cset)
        put_css_set(cset);
}
```

### Security Context Teardown Edge Cases

1. **Privileged process exit**: Special handling for processes with elevated privileges
2. **Container exit**: Namespace cleanup when containers terminate
3. **Security violation exit**: Cleanup after security policy violations
4. **Capability inheritance**: Proper cleanup to prevent privilege escalation
5. **LSM policy enforcement**: Ensuring security policies are enforced during cleanup

---

## Integration with Scheduler and Memory Management

### Scheduler Integration

The process exit implementation tightly integrates with the scheduler subsystem for proper task lifecycle management.

#### Task State Management

The scheduler tracks task states throughout the exit process:

```c
// Task state transitions during exit
current->__state = TASK_RUNNING;     // Initial state
// ... exit processing ...
current->__state = TASK_DEAD;        // Final state before removal
```

#### Final Scheduler Handoff: `do_task_dead()`

```c
void __noreturn do_task_dead(void)
{
    // Set final task state
    set_special_state(TASK_DEAD);
    
    // Remove from runqueue and schedule final context switch
    __schedule(SM_NONE);
    
    /* Never reached */
    BUG();
}
```

The scheduler's `__schedule()` function handles the `TASK_DEAD` state specially:

```c
// In kernel/sched/core.c
static void __sched notrace __schedule(unsigned int sched_mode)
{
    struct task_struct *prev, *next;
    // ... scheduling logic ...
    
    if (unlikely(prev->__state & TASK_DEAD)) {
        // Task is dead, handle final cleanup
        prev->on_cpu = 0;
        // ... additional dead task handling ...
    }
    
    // ... context switch to next task ...
}
```

#### RCU Integration

The exit path coordinates with RCU (Read-Copy-Update) for safe memory reclamation:

```c
// RCU exit coordination
exit_rcu();                    // Exit RCU subsystem
exit_tasks_rcu_start();        // Begin RCU grace period
// ... other cleanup ...
exit_tasks_rcu_finish();       // Complete RCU synchronization
```

#### CPU Affinity and Load Balancing

Process exit affects CPU load balancing:

```c
// CPU affinity cleanup during exit
void sched_exit(struct task_struct *p)
{
    unsigned long flags;
    struct rq *rq = task_rq_lock(p, &flags);
    
    // Remove task from CPU load calculations
    account_entity_dequeue(&rq->cfs, &p->se);
    
    // Update CPU load averages
    update_rq_clock(rq);
    
    task_rq_unlock(rq, p, &flags);
}
```

### Memory Management Integration

Exit processing requires coordination with the memory management subsystem for safe cleanup.

#### Memory Barriers and Synchronization

Critical memory barriers ensure proper ordering:

```c
// In exit_mm()
smp_mb__after_spinlock();  // Memory barrier for membarrier synchronization

// Memory barrier before clearing mm pointer
local_irq_disable();
current->mm = NULL;
membarrier_update_current_mm(NULL);
enter_lazy_tlb(mm, current);
local_irq_enable();
```

#### TLB and Cache Coherency

The exit path ensures TLB and cache coherency:

```c
// TLB handling during MM cleanup
mmgrab_lazy_tlb(mm);           // Grab lazy TLB reference
enter_lazy_tlb(mm, current);   // Enter lazy TLB mode
```

#### Page Fault Handling

Exiting tasks must handle potential page faults during cleanup:

```c
// Page fault handling for exiting tasks
if (unlikely(current->flags & PF_EXITING)) {
    // Special handling for exiting task page faults
    if (current->mm == NULL) {
        // No MM, handle as kernel fault
        bad_area_nosemaphore(vma, mm, code, address);
        return;
    }
}
```

#### Memory Cgroup Integration

Memory control group accounting during exit:

```c
// Memory cgroup cleanup
void mem_cgroup_exit(struct task_struct *task)
{
    struct mem_cgroup *memcg = task->memcg_in_oom;
    
    if (memcg) {
        mem_cgroup_unmark_under_oom(memcg);
        css_put(&memcg->css);
    }
    task->memcg_in_oom = NULL;
}
```

### Performance Considerations

#### Stack Usage Monitoring

The exit path includes stack usage monitoring for debugging:

```c
#ifdef CONFIG_DEBUG_STACK_USAGE
static void check_stack_usage(void)
{
    static DEFINE_SPINLOCK(low_water_lock);
    static int lowest_to_date = THREAD_SIZE;
    unsigned long free;
    
    free = stack_not_used(current);
    kstack_histogram(THREAD_SIZE - free);
    
    if (free >= lowest_to_date)
        return;
    
    spin_lock(&low_water_lock);
    if (free < lowest_to_date) {
        pr_info("%s (%d) used greatest stack depth: %lu bytes left\n",
                current->comm, task_pid_nr(current), free);
        lowest_to_date = free;
    }
    spin_unlock(&low_water_lock);
}
#endif
```

#### CPU Time Accounting

Process exit updates CPU time accounting:

```c
// CPU time accounting during exit
acct_update_integrals(tsk);    // Update accounting integrals

// Thread group CPU time accumulation
task_cputime(tsk, &utime, &stime);
sig->utime += utime;
sig->stime += stime;
sig->sum_sched_runtime += tsk->se.sum_exec_runtime;
```

#### Cache Line Optimization

The task_struct is organized to minimize cache line bouncing during exit:

```c
struct task_struct {
    // Hot fields accessed during scheduling
    volatile long __state;
    void *stack;
    
    // ... other fields ...
    
    // Exit-specific fields grouped together
    int exit_state;
    int exit_code;
    int exit_signal;
    
    // ... remaining fields ...
} ____cacheline_aligned;
```

---

## Error Handling and Edge Cases in Process Termination

### Catastrophic Failure Handling: `make_task_dead()`

The `make_task_dead()` function handles catastrophic failures that require immediate task termination:

```c
void __noreturn make_task_dead(int signr)
{
    struct task_struct *tsk = current;
    unsigned int limit;
    
    // Critical error checks
    if (unlikely(in_interrupt()))
        panic("Aiee, killing interrupt handler!");
    if (unlikely(!tsk->pid))
        panic("Attempted to kill the idle task!");
    
    // Fix critical system state issues
    if (unlikely(irqs_disabled())) {
        pr_info("note: %s[%d] exited with irqs disabled\n",
                current->comm, task_pid_nr(current));
        local_irq_enable();
    }
    
    if (unlikely(in_atomic())) {
        pr_info("note: %s[%d] exited with preempt_count %d\n",
                current->comm, task_pid_nr(current),
                preempt_count());
        preempt_count_set(PREEMPT_ENABLED);
    }
    
    // Prevent oops bombing
    limit = READ_ONCE(oops_limit);
    if (atomic_inc_return(&oops_count) >= limit && limit)
        panic("Oopsed too often (kernel.oops_limit is %d)", limit);
    
    // Handle recursive exit attempts
    if (unlikely(tsk->flags & PF_EXITING)) {
        pr_alert("Fixing recursive fault but reboot is needed!\n");
        futex_exit_recursive(tsk);
        tsk->exit_state = EXIT_DEAD;
        refcount_inc(&tsk->rcu_users);
        do_task_dead();
    }
    
    do_exit(signr);
}
```

### Oops Limit and System Stability

The kernel implements an oops limit to prevent exploitation through repeated crashes:

```c
static unsigned int oops_limit = 10000;
static atomic_t oops_count = ATOMIC_INIT(0);

// Oops counting in make_task_dead()
limit = READ_ONCE(oops_limit);
if (atomic_inc_return(&oops_count) >= limit && limit)
    panic("Oopsed too often (kernel.oops_limit is %d)", limit);
```

This mechanism prevents:
- Reference counter overflow attacks
- Resource exhaustion through repeated oopsing
- System instability from excessive error conditions

### Global Init Protection

Special protection exists for the init process (PID 1):

```c
// In do_exit()
if (group_dead) {
    if (unlikely(is_global_init(tsk)))
        panic("Attempted to kill init! exitcode=0x%08x\n",
              tsk->signal->group_exit_code ?: (int)code);
}
```

This protection ensures:
- System cannot continue without init
- Immediate panic provides clear error indication
- Prevents system hanging in unusable state

### PID Namespace Boundary Handling

Special handling for PID namespace boundaries:

```c
// PID namespace init protection
static struct task_struct *find_child_reaper(struct task_struct *father,
                                            struct list_head *dead)
{
    struct pid_namespace *pid_ns = task_active_pid_ns(father);
    struct task_struct *reaper = pid_ns->child_reaper;
    
    if (likely(reaper != father))
        return reaper;
    
    // PID namespace init is exiting - critical situation
    reaper = find_alive_thread(father);
    if (reaper) {
        pid_ns->child_reaper = reaper;
        return reaper;
    }
    
    // No suitable reaper found - namespace is dying
    write_unlock_irq(&tasklist_lock);
    
    // Clean up remaining processes
    list_for_each_entry_safe(p, n, dead, ptrace_entry) {
        list_del_init(&p->ptrace_entry);
        release_task(p);
    }
    
    // Initiate namespace cleanup
    zap_pid_ns_processes(pid_ns);
    write_lock_irq(&tasklist_lock);
    
    return father;
}
```

### Memory Management Edge Cases

#### MM Reference Counting Issues

```c
// Handle MM with no suitable owner
if (atomic_read(&mm->mm_users) <= 1) {
    WRITE_ONCE(mm->owner, NULL);
    return;
}

// Race condition handling in mm_update_next_owner()
for_each_process(g) {
    if (atomic_read(&mm->mm_users) <= 1)
        break;  // MM is going away
    if (g->flags & PF_KTHREAD)
        continue;  // Skip kernel threads
    if (try_to_set_owner(g, mm))
        goto ret;
}

// No owner found - mark as NULL
WRITE_ONCE(mm->owner, NULL);
```

#### Lazy TLB Handling

```c
// Lazy TLB reference management during exit
mmgrab_lazy_tlb(mm);  // Grab reference for lazy TLB
BUG_ON(mm != current->active_mm);

// Ensure proper TLB state transition
current->mm = NULL;
enter_lazy_tlb(mm, current);
```

### Signal Handling Edge Cases

#### Signal Delivery Race Conditions

```c
// Race-safe signal queue cleanup
void flush_sigqueue(struct sigpending *queue)
{
    struct sigqueue *q;
    
    sigemptyset(&queue->signal);
    while (!list_empty(&queue->list)) {
        q = list_entry(queue->list.next, struct sigqueue, list);
        list_del_init(&q->list);
        __sigqueue_free(q);
    }
}

// Called lockless after task removal from all lists
if (thread_group_leader(p))
    flush_sigqueue(&p->signal->shared_pending);
flush_sigqueue(&p->pending);
```

#### Group Signal Coordination

```c
// Handle signal target updates during exit
if (tsk == sig->curr_target)
    sig->curr_target = next_thread(tsk);
```

### Wait System Call Edge Cases

#### Ptrace and Parent Relationship Complexity

```c
// Complex parent-child relationship handling
if (likely(!ptrace) && unlikely(p->ptrace)) {
    // Handle ptraced child with special parent relationships
    if (!ptrace_reparented(p))
        ptrace = 1;
}

// Zombie visibility rules
if (unlikely(ptrace) || likely(!p->ptrace))
    return wait_task_zombie(wo, p);
```

#### Auto-reaping Decision Logic

```c
// Complex auto-reaping logic in exit_notify()
if (unlikely(tsk->ptrace)) {
    int sig = thread_group_leader(tsk) &&
              thread_group_empty(tsk) &&
              !ptrace_reparented(tsk) ?
              tsk->exit_signal : SIGCHLD;
    autoreap = do_notify_parent(tsk, sig);
} else if (thread_group_leader(tsk)) {
    autoreap = thread_group_empty(tsk) &&
               do_notify_parent(tsk, tsk->exit_signal);
} else {
    autoreap = true;
    do_notify_pidfd(tsk);
}
```

### Resource Cleanup Race Conditions

#### File Descriptor Cleanup Races

```c
// Race-safe file descriptor cleanup
void exit_files(struct task_struct *tsk)
{
    struct files_struct *files = tsk->files;
    
    if (files) {
        task_lock(tsk);
        tsk->files = NULL;
        task_unlock(tsk);
        put_files_struct(files);  // May block
    }
}
```

#### Task Reference Counting

```c
// RCU-protected task structure cleanup
void put_task_struct_rcu_user(struct task_struct *task)
{
    if (refcount_dec_and_test(&task->rcu_users))
        call_rcu(&task->rcu, delayed_put_task_struct);
}

static void delayed_put_task_struct(struct rcu_head *rhp)
{
    struct task_struct *tsk = container_of(rhp, struct task_struct, rcu);
    
    kprobe_flush_task(tsk);
    rethook_flush_task(tsk);
    perf_event_delayed_put(tsk);
    trace_sched_process_free(tsk);
    put_task_struct(tsk);
}
```

### Debugging and Monitoring

#### Stack Usage Validation

```c
#ifdef CONFIG_DEBUG_STACK_USAGE
static void check_stack_usage(void)
{
    unsigned long free = stack_not_used(current);
    
    if (free < lowest_to_date) {
        pr_info("%s (%d) used greatest stack depth: %lu bytes left\n",
                current->comm, task_pid_nr(current), free);
        lowest_to_date = free;
    }
}
#endif
```

#### Lock Validation

```c
// Ensure no locks are held during exit
debug_check_no_locks_held();

// Lockdep cleanup
lockdep_free_task(tsk);
```

---

## System Call Interface

### Primary System Calls

The kernel provides several system calls for process termination and waiting:

#### `exit()` System Call

```c
SYSCALL_DEFINE1(exit, int, error_code)
{
    do_exit((error_code & 0xff) << 8);
}
```

#### `exit_group()` System Call

```c
SYSCALL_DEFINE1(exit_group, int, error_code)
{
    do_group_exit((error_code & 0xff) << 8);
    return 0;  // Never reached
}
```

#### `wait4()` System Call

```c
SYSCALL_DEFINE4(wait4, pid_t, upid, int __user *, stat_addr,
                int, options, struct rusage __user *, ru)
{
    struct rusage r;
    long err = kernel_wait4(upid, stat_addr, options, ru ? &r : NULL);
    
    if (err > 0) {
        if (ru && copy_to_user(ru, &r, sizeof(struct rusage)))
            return -EFAULT;
    }
    return err;
}
```

#### `waitid()` System Call

```c
SYSCALL_DEFINE5(waitid, int, which, pid_t, upid, struct siginfo __user *,
                infop, int, options, struct rusage __user *, ru)
{
    struct rusage r;
    struct waitid_info info = {.status = 0};
    long err = kernel_waitid(which, upid, &info, options, ru ? &r : NULL);
    int signo = 0;
    
    if (err > 0) {
        signo = SIGCHLD;
        err = 0;
        if (ru && copy_to_user(ru, &r, sizeof(struct rusage)))
            return -EFAULT;
    }
    
    if (!infop)
        return err;
    
    // Copy extended wait information to user space
    if (!user_write_access_begin(infop, sizeof(*infop)))
        return -EFAULT;
    
    unsafe_put_user(signo, &infop->si_signo, Efault);
    unsafe_put_user(0, &infop->si_errno, Efault);
    unsafe_put_user(info.cause, &infop->si_code, Efault);
    unsafe_put_user(info.pid, &infop->si_pid, Efault);
    unsafe_put_user(info.uid, &infop->si_uid, Efault);
    unsafe_put_user(info.status, &infop->si_status, Efault);
    user_write_access_end();
    return err;
    
Efault:
    user_write_access_end();
    return -EFAULT;
}
```

### Kernel-Internal Wait Functions

#### `kernel_wait4()`

```c
long kernel_wait4(pid_t upid, int __user *stat_addr, int options,
                  struct rusage *ru)
{
    struct wait_opts wo;
    struct pid *pid = NULL;
    enum pid_type type;
    long ret;
    
    // Validate options
    if (options & ~(WNOHANG|WUNTRACED|WCONTINUED|
                    __WNOTHREAD|__WCLONE|__WALL))
        return -EINVAL;
    
    // Handle special PID values
    if (upid == -1)
        type = PIDTYPE_MAX;        // Wait for any child
    else if (upid < 0) {
        type = PIDTYPE_PGID;       // Wait for process group
        pid = find_get_pid(-upid);
    } else if (upid == 0) {
        type = PIDTYPE_PGID;       // Wait for same process group
        pid = get_task_pid(current, PIDTYPE_PGID);
    } else {
        type = PIDTYPE_PID;        // Wait for specific process
        pid = find_get_pid(upid);
    }
    
    // Initialize wait options
    wo.wo_type      = type;
    wo.wo_pid       = pid;
    wo.wo_flags     = options | WEXITED;
    wo.wo_info      = NULL;
    wo.wo_stat      = 0;
    wo.wo_rusage    = ru;
    
    ret = do_wait(&wo);
    put_pid(pid);
    
    // Copy status to user space
    if (ret > 0 && stat_addr && put_user(wo.wo_stat, stat_addr))
        ret = -EFAULT;
    
    return ret;
}
```

#### `kernel_wait()`

```c
int kernel_wait(pid_t pid, int *stat)
{
    struct wait_opts wo = {
        .wo_type        = PIDTYPE_PID,
        .wo_pid         = find_get_pid(pid),
        .wo_flags       = WEXITED,
    };
    int ret;
    
    ret = do_wait(&wo);
    if (ret > 0 && wo.wo_stat)
        *stat = wo.wo_stat;
    put_pid(wo.wo_pid);
    return ret;
}
```

### Wait Option Flags

The wait system calls support various option flags:

```c
// Standard POSIX wait options
#define WNOHANG     0x00000001  // Don't block if no child available
#define WUNTRACED   0x00000002  // Report stopped children
#define WSTOPPED    WUNTRACED   // Synonym for WUNTRACED
#define WEXITED     0x00000004  // Report exited children
#define WCONTINUED  0x00000008  // Report continued children
#define WNOWAIT     0x01000000  // Don't reap, just poll

// Linux-specific extensions
#define __WNOTHREAD 0x20000000  // Don't wait on children of other threads
#define __WCLONE    0x80000000  // Wait only on non-SIGCHLD children
#define __WALL      0x40000000  // Wait on all children regardless
```

### PID Types and Targeting

The wait system supports different PID targeting modes:

```c
enum pid_type {
    PIDTYPE_PID,    // Specific process ID
    PIDTYPE_TGID,   // Thread group ID (main thread)
    PIDTYPE_PGID,   // Process group ID
    PIDTYPE_SID,    // Session ID
    PIDTYPE_MAX     // Special: any child
};
```

### Status Code Interpretation

Wait system calls return status codes that encode exit information:

```c
// Status code macros (from user space perspective)
#define WEXITSTATUS(status)     (((status) & 0xff00) >> 8)
#define WTERMSIG(status)        ((status) & 0x7f)
#define WSTOPSIG(status)        WEXITSTATUS(status)
#define WIFEXITED(status)       (WTERMSIG(status) == 0)
#define WIFSIGNALED(status)     (!WIFEXITED(status) && !WIFSTOPPED(status))
#define WIFSTOPPED(status)      (((status) & 0xff) == 0x7f)
#define WIFCONTINUED(status)    ((status) == 0xffff)
#define WCOREDUMP(status)       ((status) & 0x80)
```

### Extended Wait Information

The `waitid()` system call provides extended information through `siginfo_t`:

```c
struct waitid_info {
    pid_t pid;      // Child process ID
    uid_t uid;      // Child real user ID
    int status;     // Exit status or signal number
    int cause;      // Reason for child state change
};

// Cause codes
#define CLD_EXITED      1   // Child has exited
#define CLD_KILLED      2   // Child was killed
#define CLD_DUMPED      3   // Child terminated abnormally
#define CLD_TRAPPED     4   // Traced child has trapped
#define CLD_STOPPED     5   // Child has stopped
#define CLD_CONTINUED   6   // Stopped child has continued
```

---

## Data Structures Reference

### Core Task Structure

```c
struct task_struct {
    // Task state and scheduling
    volatile long           __state;
    void                   *stack;
    refcount_t             usage;
    unsigned int           flags;
    unsigned int           ptrace;
    int                    on_cpu;
    struct wake_q_node     wake_q;
    int                    wake_cpu;
    
    // Process identification
    pid_t                  pid;
    pid_t                  tgid;
    struct task_struct     *real_parent;
    struct task_struct     *parent;
    struct list_head       children;
    struct list_head       sibling;
    struct task_struct     *group_leader;
    
    // Exit state
    int                    exit_state;
    int                    exit_code;
    int                    exit_signal;
    int                    pdeath_signal;
    
    // Memory management
    struct mm_struct       *mm;
    struct mm_struct       *active_mm;
    
    // Filesystem
    struct fs_struct       *fs;
    struct files_struct    *files;
    
    // Namespaces
    struct nsproxy         *nsproxy;
    
    // Signal handling
    struct signal_struct   *signal;
    struct sighand_struct  *sighand;
    struct sigpending      pending;
    
    // Credentials
    const struct cred      *real_cred;
    const struct cred      *cred;
    
    // ... many other fields ...
};
```

### Signal Structure

```c
struct signal_struct {
    refcount_t             sigcnt;
    atomic_t               live;
    int                    nr_threads;
    int                    quick_threads;
    struct list_head       thread_head;
    
    wait_queue_head_t      wait_chldexit;
    struct task_struct     *curr_target;
    struct sigpending      shared_pending;
    
    // Thread group exit
    int                    group_exit_code;
    int                    notify_count;
    struct task_struct     *group_exec_task;
    int                    group_stop_count;
    unsigned int           flags;
    
    struct core_state      *core_state;
    
    // Child subreaper support
    unsigned int           is_child_subreaper:1;
    unsigned int           has_child_subreaper:1;
    
    // Statistics and accounting
    seqlock_t              stats_lock;
    u64                    utime, stime, cutime, cstime;
    u64                    gtime, cgtime;
    unsigned long          nvcsw, nivcsw, cnvcsw, cnivcsw;
    unsigned long          min_flt, maj_flt, cmin_flt, cmaj_flt;
    unsigned long          inblock, oublock, cinblock, coublock;
    unsigned long          maxrss, cmaxrss;
    
    // Process group and session
    struct pid             *pids[PIDTYPE_MAX];
    struct tty_struct      *tty;
    
    // ... additional fields ...
};
```

### Signal Handler Structure

```c
struct sighand_struct {
    spinlock_t             siglock;
    refcount_t             count;
    wait_queue_head_t      signalfd_wqh;
    struct k_sigaction     action[_NSIG];
};
```

### Wait Options Structure

```c
struct wait_opts {
    enum pid_type          wo_type;
    int                    wo_flags;
    struct pid            *wo_pid;
    struct waitid_info    *wo_info;
    int                    wo_stat;
    struct rusage         *wo_rusage;
    wait_queue_entry_t     child_wait;
    int                    notask_error;
};
```

### Release Task Post Structure

```c
struct release_task_post {
    struct pid            *pids[PIDTYPE_MAX];
};
```

### Core State for Coredumps

```c
struct core_state {
    atomic_t               nr_threads;
    struct core_thread     dumper;
    struct completion      startup;
};

struct core_thread {
    struct task_struct    *task;
    struct core_thread    *next;
};
```

### Memory Management Structure

```c
struct mm_struct {
    struct {
        struct vm_area_struct *mmap;
        struct rb_root         mm_rb;
        u64                    vmacache_seqnum;
        unsigned long          task_size;
        unsigned long          highest_vm_end;
        pgd_t                 *pgd;
        
        atomic_t              mm_users;
        atomic_t              mm_count;
        
        struct list_head      mmlist;
        unsigned long         hiwater_rss;
        unsigned long         hiwater_vm;
        unsigned long         total_vm;
        unsigned long         locked_vm;
        
        unsigned long         start_code, end_code;
        unsigned long         start_data, end_data;
        unsigned long         start_brk, brk;
        unsigned long         start_stack;
        
        unsigned long         arg_start, arg_end;
        unsigned long         env_start, env_end;
        
        struct task_struct   *owner;
        
        // ... many more fields ...
    } ____cacheline_aligned_in_smp;
};
```

### Namespace Proxy Structure

```c
struct nsproxy {
    atomic_t                count;
    struct uts_namespace   *uts_ns;
    struct ipc_namespace   *ipc_ns;
    struct mnt_namespace   *mnt_ns;
    struct pid_namespace   *pid_ns;
    struct net            *net_ns;
    struct time_namespace  *time_ns;
    struct cgroup_namespace *cgroup_ns;
};
```

### PID Structure

```c
struct pid {
    refcount_t             count;
    unsigned int           level;
    spinlock_t             lock;
    struct hlist_head      tasks[PIDTYPE_MAX];
    struct hlist_head      inodes;
    wait_queue_head_t      wait_pidfd;
    struct rcu_head        rcu;
    struct upid            numbers[1];
};
```

This comprehensive documentation covers the Linux kernel's process exit implementation from multiple perspectives, providing detailed analysis of the core exit flow, memory management cleanup, parent-child relationships, thread group coordination, namespace cleanup, scheduler integration, error handling, system call interfaces, and key data structures. The implementation demonstrates the complexity required to safely and efficiently terminate processes while maintaining system integrity and consistency.