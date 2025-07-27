# kernel/locking/mutex.c - Linux Kernel Mutex Implementation

## Overview

This file implements the high-performance mutex synchronization primitive for the Linux kernel, providing exclusive access control with sophisticated optimizations for both single-core and multi-core systems. Originally developed as a replacement for semaphores in scenarios requiring mutual exclusion, the Linux mutex has evolved into one of the most advanced and efficient locking primitives in any operating system, featuring optimistic spinning, MCS queuing, adaptive behavior, and comprehensive debugging support.

## Historical Development

### Key Evolution Points
- **Early Linux (1991-2003)**: Simple semaphore-based locking
- **Mutex Introduction (2006)**: Ingo Molnar's initial mutex implementation
- **Optimistic Spinning (2009)**: Peter Zijlstra's spinning optimization
- **MCS Queuing (2013)**: Jason Low's scalability improvements  
- **RT-Mutex Integration (2015)**: Real-time priority inheritance support
- **Modern Era (2016-present)**: Wound-wait mutexes, guard macros, and adaptive handoff

### Design Philosophy
The mutex implementation embodies the principle of "fast path optimization with fairness guarantees," where the common case (uncontended access) requires minimal overhead while heavily contended scenarios maintain fairness through sophisticated queuing and handoff mechanisms. The design emphasizes adaptive behavior, automatically switching between spinning and sleeping based on system conditions.

## Core Architecture

### Locking Strategy Hierarchy
```
Fast Path (Uncontended) → Optimistic Spinning → Slow Path (Blocking)
         ↓                        ↓                    ↓
[Single Atomic CAS]    [MCS Queue + Owner Spin]  [FIFO Wait Queue]
```

### State Machine Model
```
UNLOCKED → LOCKED → CONTENDED → HANDOFF → PICKUP → UNLOCKED
    ↓        ↓         ↓           ↓         ↓         ↓
[Available] [Owned]  [Waiters]  [Transfer] [Acquired] [Released]
```

### Dual Implementation Strategy
```
Non-RT Configuration: Optimized Mutex with Spinning
RT Configuration: RT-Mutex with Priority Inheritance
```

## Key Data Structures

### Core Mutex Structure
```c
struct mutex {
    atomic_long_t               owner;      /* Owner task + flags */
    raw_spinlock_t              wait_lock;  /* Protects wait_list */
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    struct optimistic_spin_queue osq;       /* MCS lock for spinners */
#endif
    struct list_head            wait_list;  /* FIFO queue of waiters */
#ifdef CONFIG_DEBUG_MUTEXES
    void                       *magic;      /* Debug validation */
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map          dep_map;    /* Lockdep integration */
#endif
} __MUTEX_INITIALIZER(lockname);
```

### Owner Field Encoding
```c
/* Packed into atomic_long_t owner field */
#define MUTEX_FLAG_WAITERS      0x01    /* Non-empty waiter list */
#define MUTEX_FLAG_HANDOFF      0x02    /* Handoff to first waiter */
#define MUTEX_FLAG_PICKUP       0x04    /* Handoff completed */

/* Bits [63:3] or [31:3]: Task pointer (NULL = unlocked) */
/* Bits [2:0]: Status flags */
```

### Waiter Structure
```c
struct mutex_waiter {
    struct list_head            list;       /* Queue linkage */
    struct task_struct         *task;       /* Blocked task */
    struct ww_acquire_ctx      *ww_ctx;     /* Wound-wait context */
#ifdef CONFIG_DEBUG_MUTEXES
    void                       *magic;      /* Debug validation */
#endif
};
```

### Optimistic Spin Queue
```c
struct optimistic_spin_queue {
    atomic_t                    tail;       /* Encoded CPU of tail spinner */
};

struct optimistic_spin_node {
    struct optimistic_spin_node *next;      /* Next spinner in queue */
    struct optimistic_spin_node *prev;      /* Previous spinner */
    int                         locked;     /* Node lock status */
    int                         cpu;        /* CPU identifier */
};
```

### RT-Mutex Integration (CONFIG_PREEMPT_RT)
```c
#ifdef CONFIG_PREEMPT_RT
struct mutex {
    struct rt_mutex_base        rtmutex;    /* RT-mutex base */
    struct lockdep_map          dep_map;    /* Lockdep integration */
};
#endif
```

## Core Functions

### Fast Path Operations

#### `mutex_lock()` - Primary Locking Interface
```c
void __sched mutex_lock(struct mutex *lock)
```

**Purpose**: Acquire exclusive access to protected resource

**Fast Path Algorithm**:
```c
void __sched mutex_lock(struct mutex *lock)
{
    might_sleep();
    
    if (!__mutex_trylock_fast(lock))
        __mutex_lock_slowpath(lock);
}

static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;
    unsigned long zero = 0UL;
    
    return atomic_long_try_cmpxchg_acquire(&lock->owner, &zero, curr);
}
```

**Performance Characteristics**:
- **Uncontended case**: Single atomic compare-and-swap operation
- **Memory ordering**: Acquire semantics ensure proper happens-before relationships
- **Branch prediction**: Fast path optimized for common uncontended case

#### `mutex_unlock()` - Release Operation
```c
void __sched mutex_unlock(struct mutex *lock)
```

**Fast Path Implementation**:
```c
void __sched mutex_unlock(struct mutex *lock)
{
#ifndef CONFIG_DEBUG_LOCK_ALLOC
    if (__mutex_unlock_fast(lock))
        return;
#endif
    __mutex_unlock_slowpath(lock, _RET_IP_);
}

static __always_inline bool __mutex_unlock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;
    return atomic_long_try_cmpxchg_release(&lock->owner, &curr, 0UL);
}
```

**Release Semantics**:
- **Memory ordering**: Release semantics ensure critical section visibility
- **Owner validation**: Verifies current task owns the mutex
- **Flag checking**: Slow path handles waiters and handoff flags

### Optimistic Spinning Infrastructure

#### `mutex_optimistic_spin()` - Adaptive Spinning
```c
static bool mutex_optimistic_spin(struct mutex *lock, struct ww_acquire_ctx *ww_ctx,
                                  struct mutex_waiter *waiter)
```

**Two-Level Spinning Architecture**:

**Level 1: MCS Queue Participation**
```c
if (!osq_lock(&lock->osq))
    goto fail;
    
/* MCS queue ensures only one task spins on the lock owner */
```

**Level 2: Owner Field Spinning**
```c
for (;;) {
    owner = __mutex_trylock_or_owner(lock);
    if (!owner)
        break;  /* Lock acquired */
        
    if (!mutex_spin_on_owner(lock, owner, ww_ctx, waiter))
        goto fail_unlock;
        
    cpu_relax();
}
```

**Spinning Heuristics**:
1. **Owner running check**: Spin only if owner is actively running
2. **Preemption detection**: Exit if scheduler requests reschedule
3. **Virtualization awareness**: Handle VCPU preemption gracefully
4. **Adaptive timeout**: Bounded spinning to prevent live-lock

#### `mutex_spin_on_owner()` - Owner State Monitoring
```c
static noinline bool mutex_spin_on_owner(struct mutex *lock, struct task_struct *owner,
                                         struct ww_acquire_ctx *ww_ctx, 
                                         struct mutex_waiter *waiter)
```

**Termination Conditions**:
```c
static noinline bool mutex_spin_on_owner(struct mutex *lock, struct task_struct *owner,
                                         struct ww_acquire_ctx *ww_ctx,
                                         struct mutex_waiter *waiter)
{
    bool ret = true;
    
    rcu_read_lock();
    while (__mutex_owner(lock) == owner) {
        barrier();
        
        if (!owner_on_cpu(owner) || need_resched()) {
            ret = false;
            break;
        }
        
        if (ww_ctx && __ww_mutex_check_kill(lock, ww_ctx, waiter)) {
            ret = false;
            break;
        }
        
        cpu_relax();
    }
    rcu_read_unlock();
    
    return ret;
}
```

### Slow Path and Wait Queue Management

#### `__mutex_lock_slowpath()` - Contended Case Handler
```c
static int __sched __mutex_lock_common(struct mutex *lock, unsigned int state,
                                       unsigned int subclass,
                                       struct lockdep_map *nest_lock,
                                       unsigned long ip,
                                       struct ww_acquire_ctx *ww_ctx,
                                       const bool use_ww_ctx)
```

**Slow Path Process**:
1. **Waiter setup**: Initialize waiter structure and add to wait queue
2. **Optimistic spinning**: First waiter can spin without OSQ overhead
3. **Blocking**: Set task state and schedule if spinning fails
4. **Signal handling**: Check for interruption in interruptible variants
5. **Wakeup handling**: Process wakeup and handoff mechanisms

**Waiter Queue Management**:
```c
static void __mutex_add_waiter(struct mutex *lock, struct mutex_waiter *waiter,
                               struct list_head *list)
{
    debug_mutex_add_waiter(lock, waiter, current);
    
    list_add_tail(&waiter->list, list);
    if (__mutex_waiter_is_first(lock, waiter))
        __mutex_set_flag(lock, MUTEX_FLAG_WAITERS);
}
```

#### Fairness and Handoff Mechanism
```c
static void __mutex_handoff(struct mutex *lock, struct task_struct *task)
{
    unsigned long owner = atomic_long_read(&lock->owner);
    
    for (;;) {
        unsigned long new = (owner & MUTEX_FLAG_WAITERS);
        new |= (unsigned long)task;
        if (task)
            new |= MUTEX_FLAG_PICKUP;
            
        if (atomic_long_try_cmpxchg_release(&lock->owner, &owner, new))
            break;
    }
}
```

**Handoff Process**:
1. **Trigger condition**: Multiple failed spinning attempts
2. **Direct transfer**: Lock ownership transferred to first waiter
3. **Pickup protocol**: Waiter validates handoff completion
4. **Starvation prevention**: Ensures bounded waiting times

### API Variants and Error Handling

#### Interruptible Lock Variants
```c
int __sched mutex_lock_interruptible(struct mutex *lock)
{
    might_sleep();
    
    if (__mutex_trylock_fast(lock))
        return 0;
        
    return __mutex_lock_interruptible_slowpath(lock);
}

int __sched mutex_lock_killable(struct mutex *lock)
{
    might_sleep();
    
    if (__mutex_trylock_fast(lock))
        return 0;
        
    return __mutex_lock_killable_slowpath(lock);
}
```

**Signal Handling**:
```c
/* In slow path */
if (signal_pending_state(state, current)) {
    ret = -EINTR;
    goto err;
}
```

#### Non-blocking Trylock
```c
int __sched mutex_trylock(struct mutex *lock)
{
    MUTEX_WARN_ON(lock->magic != lock);
    return __mutex_trylock(lock);
}
```

**Return Convention**: 1 on success, 0 on contention (follows spinlock convention)

### Debugging and Validation Infrastructure

#### Debug Mutex Extensions
```c
#ifdef CONFIG_DEBUG_MUTEXES
struct mutex_waiter {
    /* ... standard fields ... */
    void                       *magic;      /* Self-reference for validation */
};

#define MUTEX_DEBUG_INIT        0x11
#define MUTEX_DEBUG_FREE        0x22
#define MUTEX_POISON_WW_CTX     0x33
#endif
```

#### Lockdep Integration
```c
static inline void mutex_acquire_nest(struct lockdep_map *lock,
                                      unsigned int subclass, int trylock,
                                      struct lockdep_map *nest_lock,
                                      unsigned long ip)
{
    lock_acquire_exclusive(lock, subclass, trylock, nest_lock, ip);
}
```

**Debug Features**:
- **Magic number validation**: Detects corruption and use-after-free
- **Owner tracking**: Validates proper ownership during unlock
- **Waiter consistency**: Ensures proper wait list management
- **Deadlock detection**: Integration with lockdep dependency tracking

#### Performance Monitoring
```c
/* Trace points for performance analysis */
static inline void trace_contention_begin(struct mutex *lock, unsigned int flags)
{
    trace_contention_begin(lock, flags | LCB_F_MUTEX);
}

static inline void trace_contention_end(struct mutex *lock, int ret)
{
    trace_contention_end(lock, ret);
}
```

### RT-Mutex Integration

#### RT Configuration Override
```c
#ifdef CONFIG_PREEMPT_RT
/* All mutex operations delegate to RT-mutex implementation */
void __sched mutex_lock(struct mutex *lock)
{
    rtmutex_lock(&lock->rtmutex);
}

void __sched mutex_unlock(struct mutex *lock)
{
    rtmutex_unlock(&lock->rtmutex);
}
#endif
```

**RT Features**:
- **Priority inheritance**: Prevents priority inversion
- **Deterministic behavior**: Bounded execution times
- **Real-time scheduling**: Integration with RT scheduler

### Wound-Wait Mutex Extensions

#### Deadlock Avoidance
```c
static bool __ww_mutex_wound(struct mutex *lock, struct ww_acquire_ctx *ww_ctx,
                             struct ww_acquire_ctx *hold_ctx)
{
    if (ww_ctx->stamp - hold_ctx->stamp <= LONG_MAX &&
        (ww_ctx->stamp != hold_ctx->stamp || ww_ctx > hold_ctx)) {
        hold_ctx->wounded = 1;
        wake_up_process(hold_ctx->task);
        return true;
    }
    return false;
}
```

**Wound-Wait Algorithm**:
- **Timestamp ordering**: Prevents circular dependencies
- **Wound mechanism**: Older contexts force younger ones to back off
- **Deadlock avoidance**: Guarantees forward progress in complex locking scenarios

## Advanced Features

### Performance Optimizations

#### Cache-Friendly Design
```c
/* Per-CPU OSQ nodes prevent false sharing */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct optimistic_spin_node, osq_node);

/* Compact mutex structure fits in single cache line */
struct mutex {
    atomic_long_t       owner;      /* 8 bytes with embedded flags */
    raw_spinlock_t      wait_lock;  /* Minimal spinlock */
    /* ... additional fields conditionally compiled */
};
```

#### Memory Ordering Optimizations
```c
/* Precise acquire/release semantics */
atomic_long_try_cmpxchg_acquire(&lock->owner, &zero, curr);  /* Lock */
atomic_long_try_cmpxchg_release(&lock->owner, &curr, 0UL);   /* Unlock */
```

#### Adaptive Behavior
- **Load-sensitive spinning**: Adjusts behavior based on system load
- **NUMA awareness**: Per-CPU data structures minimize remote memory access
- **Virtualization support**: Handles VCPU preemption gracefully

### Guard Macros and Modern C++ Style RAII
```c
#define DEFINE_GUARD(name, type, lock, unlock)                          \
    DEFINE_CLASS(name, type, unlock, ({ lock; _T; }), type _T)

/* Automatic unlock on scope exit */
void critical_section(void)
{
    guard(mutex)(&my_mutex);
    /* Critical section code */
    /* Automatic unlock when guard goes out of scope */
}
```

## Integration Points

### Scheduler Integration
- **might_sleep()**: Static analysis annotation for blocking behavior
- **need_resched()**: Integration with preemption and scheduling
- **owner_on_cpu()**: Coordination with scheduler state tracking

### Memory Management Integration
- **RCU protection**: Safe access to task structures during spinning
- **Memory barriers**: Proper ordering with memory management operations
- **NUMA considerations**: Optimized for multi-socket systems

### Power Management Integration
- **CPU idle**: Efficient blocking that allows CPU idle states
- **Frequency scaling**: Integration with DVFS through scheduler hints

### Virtualization Integration
- **Paravirtualization**: Guest-aware spinning behavior
- **VCPU preemption**: Detection and handling of hypervisor scheduling

This comprehensive mutex implementation represents the pinnacle of synchronization primitive engineering, providing excellent performance across diverse workloads while maintaining strong fairness guarantees and comprehensive debugging support suitable for production kernel code.