# kernel/sched/core.c - Linux CPU Scheduler Core Implementation

## Overview

This file implements the core CPU scheduler for the Linux kernel, originally designed by Linus Torvalds and extensively developed by Ingo Molnar. It provides the fundamental task scheduling infrastructure that manages CPU time allocation among processes and threads, handling preemption, context switching, load balancing, and multi-core scheduling coordination. The scheduler is central to system performance and fairness, determining which tasks run when and for how long.

## Historical Development

### Key Contributors and Milestones
- **Linus Torvalds (1991-2002)**: Original scheduler design and early implementations
- **Ingo Molnar (1998-2024)**: Major scheduler redesigns, O(1) scheduler, CFS development
- **Red Hat Inc.**: Corporate sponsorship and development resources
- **Community Contributors**: Thousands of developers contributing optimizations and features

### Major Scheduler Evolution
- **1991-1996**: Simple round-robin scheduler
- **2001**: O(1) scheduler by Ingo Molnar - constant time operations
- **2007**: Completely Fair Scheduler (CFS) - replaced O(1) scheduler
- **2012**: RT scheduling classes and real-time improvements
- **2020s**: Core scheduling, deadline scheduling, and modern optimizations

## Core Concepts

### Scheduling Architecture

#### Multi-Class Scheduler Design
The Linux scheduler uses a hierarchical class-based design:

```
┌─────────────────────────────────────────┐
│            stop_sched_class             │  ← Highest Priority
├─────────────────────────────────────────┤
│             dl_sched_class              │  ← Deadline Scheduling
├─────────────────────────────────────────┤
│             rt_sched_class              │  ← Real-Time Scheduling
├─────────────────────────────────────────┤
│            fair_sched_class             │  ← Normal Tasks (CFS)
├─────────────────────────────────────────┤
│            idle_sched_class             │  ← Idle Tasks
└─────────────────────────────────────────┘
```

#### Scheduling Policies
- **SCHED_NORMAL**: Default time-sharing policy (CFS)
- **SCHED_BATCH**: Background batch processing
- **SCHED_IDLE**: Very low priority tasks
- **SCHED_FIFO**: Real-time FIFO policy
- **SCHED_RR**: Real-time round-robin policy
- **SCHED_DEADLINE**: Deadline-sensitive real-time tasks

### Task States and Transitions
```
    TASK_RUNNING ←→ TASK_INTERRUPTIBLE
         ↕              ↓
    [Running] ←→ TASK_UNINTERRUPTIBLE
         ↓              ↓
    TASK_STOPPED → TASK_ZOMBIE → TASK_DEAD
```

## Key Data Structures

### `struct rq` - Per-CPU Run Queue
```c
struct rq {
    raw_spinlock_t          __lock;         /* Run queue lock */
    unsigned int            nr_running;     /* Number of runnable tasks */
    unsigned int            nr_numa_running; /* NUMA tasks */
    unsigned int            nr_preferred_running; /* Preferred tasks */
    
    struct cfs_rq           cfs;            /* CFS run queue */
    struct rt_rq            rt;             /* RT run queue */
    struct dl_rq            dl;             /* Deadline run queue */
    
    struct task_struct __rcu *curr;         /* Currently running task */
    struct task_struct      *idle;         /* Idle task */
    struct task_struct      *stop;         /* Stop task */
    
    u64                     clock;          /* Run queue clock */
    u64                     clock_task;     /* Task clock */
    u64                     clock_pelt;     /* PELT clock */
    u64                     clock_pelt_idle; /* PELT idle clock */
    
    unsigned long           nr_uninterruptible; /* Uninterruptible tasks */
    int                     active_balance; /* Active balancing flag */
    int                     cpu;            /* CPU number */
    int                     online;         /* CPU online status */
    
    struct root_domain      *rd;            /* Root domain */
    struct sched_domain __rcu *sd;          /* Scheduling domains */
    
    unsigned long           nr_switches;    /* Context switch count */
    unsigned long           nr_migrations_in; /* Migration count */
    
    struct balance_callback *balance_callback; /* Load balancing */
    
    /* Core scheduling support */
#ifdef CONFIG_SCHED_CORE
    struct rq               *core;          /* Core run queue */
    struct task_struct      *core_pick;     /* Core-wide pick */
    unsigned int            core_enabled;   /* Core scheduling enabled */
    unsigned int            core_sched_seq; /* Core scheduling sequence */
    struct rb_root          core_tree;     /* Core scheduling tree */
    unsigned char           core_cookie;    /* Core scheduling cookie */
    unsigned int            core_forceidle_count; /* Force idle count */
    unsigned int            core_forceidle_seq;   /* Force idle sequence */
    unsigned int            core_forceidle_occupation; /* Force idle occupation */
    u64                     core_forceidle_start; /* Force idle start time */
#endif

    /* Load averaging and tracking */
    struct load_weight      load;           /* Run queue load */
    unsigned long           nr_load_updates; /* Load update count */
    u64                     load_avg;       /* Load average */
    
    /* Power and thermal management */
    unsigned int            nr_capacity_calls; /* Capacity calls */
    u64                     avg_rt;         /* Average RT utilization */
    u64                     avg_dl;         /* Average deadline utilization */
    u64                     avg_irq;        /* Average IRQ time */
    
    /* Bandwidth control */
    u64                     rt_avg;         /* RT average */
    unsigned long           rt_runtime;     /* RT runtime */
    
    /* Thermal pressure */
    unsigned long           thermal_pressure; /* Thermal pressure */
    
    /* Frequency scaling */
    struct cpufreq_policy   *freq_policy;   /* Frequency policy */
    
    /* Statistics and debugging */
    unsigned int            ttwu_count;     /* Wake-up count */
    unsigned int            ttwu_local;     /* Local wake-ups */
    
    #ifdef CONFIG_SMP
    struct callback_head    *balance_push_callback; /* Push callback */
    struct cpu_stop_work    balance_work;   /* Balance work */
    #endif
};
```

### `struct sched_class` - Scheduling Class Operations
```c
struct sched_class {
    int uclamp_enabled;                     /* Utilization clamping */
    
    void (*enqueue_task)(struct rq *rq, struct task_struct *p, int flags);
    void (*dequeue_task)(struct rq *rq, struct task_struct *p, int flags);
    void (*yield_task)(struct rq *rq);
    bool (*yield_to_task)(struct rq *rq, struct task_struct *p);
    
    void (*wakeup_preempt)(struct rq *rq, struct task_struct *p, int flags);
    
    struct task_struct *(*pick_next_task)(struct rq *rq, struct task_struct *prev);
    struct task_struct *(*pick_task)(struct rq *rq);
    void (*put_prev_task)(struct rq *rq, struct task_struct *p);
    void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);
    
    int (*balance)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
    int (*select_task_rq)(struct task_struct *p, int task_cpu, int flags);
    void (*migrate_task_rq)(struct task_struct *p, int new_cpu);
    
    void (*task_woken)(struct rq *rq, struct task_struct *task);
    
    void (*set_cpus_allowed)(struct task_struct *p, struct cpumask *newmask, u32 flags);
    
    void (*rq_online)(struct rq *rq);
    void (*rq_offline)(struct rq *rq);
    
    struct rq *(*find_lock_rq)(struct task_struct *p, struct rq *rq);
    
    void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
    void (*task_fork)(struct task_struct *p);
    void (*task_dead)(struct task_struct *p);
    
    void (*switched_from)(struct rq *rq, struct task_struct *p);
    void (*switched_to)(struct rq *rq, struct task_struct *p);
    void (*prio_changed)(struct rq *rq, struct task_struct *p, int oldprio);
    
    unsigned int (*get_rr_interval)(struct rq *rq, struct task_struct *task);
    
    void (*update_curr)(struct rq *rq);
    
    void (*task_change_group)(struct task_struct *p);
    
    #ifdef CONFIG_FAIR_GROUP_SCHED
    void (*task_move_group)(struct task_struct *p);
    #endif
    
    #ifdef CONFIG_UCLAMP_TASK
    void (*uclamp_enabled)(void);
    #endif
    
    #ifdef CONFIG_SCHED_CORE
    int (*task_is_throttled)(struct task_struct *p, int cpu);
    #endif
};
```

### Task States
```c
#define TASK_RUNNING            0x00000000  /* Task is running or runnable */
#define TASK_INTERRUPTIBLE      0x00000001  /* Task sleeping, can be woken by signals */
#define TASK_UNINTERRUPTIBLE    0x00000002  /* Task sleeping, cannot be woken by signals */
#define __TASK_STOPPED          0x00000004  /* Task stopped by debugger or job control */
#define __TASK_TRACED           0x00000008  /* Task being traced by ptrace */
#define TASK_IDLE               0x00000040  /* Task is an idle task */
#define TASK_NEW                0x00000080  /* Task is being created */
#define TASK_WAKEKILL           0x00000100  /* Task can be killed while sleeping */
#define TASK_WAKING             0x00000200  /* Task is waking up */
#define TASK_PARKED             0x00000400  /* Task is parked by kthread_park() */
#define TASK_NOLOAD             0x00000800  /* Task doesn't contribute to load average */
#define TASK_FROZEN             0x00001000  /* Task is frozen for suspend */
#define TASK_STATE_MAX          0x00002000  /* Maximum task state value */
```

## Core Functions

### Main Scheduling Function

#### `__schedule()` - Core Scheduler Entry Point
```c
static void __sched notrace __schedule(int sched_mode)
```

**Purpose**: Main scheduler function that performs task selection and context switching

**Entry Points**:
1. **Explicit Blocking**: mutex, semaphore, waitqueue operations
2. **Preemption**: TIF_NEED_RESCHED flag checked on interrupts and userspace returns
3. **Voluntary**: Direct schedule() calls and cond_resched()

**Scheduling Process**:
1. **Setup and Locking**: Acquire run queue lock, disable interrupts
2. **State Management**: Handle task state changes and blocking
3. **Task Selection**: Pick next task using scheduling classes
4. **Context Switch**: Switch between tasks if different
5. **Cleanup**: Update statistics, handle callbacks

**Key Operations**:
```c
cpu = smp_processor_id();
rq = cpu_rq(cpu);
prev = rq->curr;

rq_lock(rq, &rf);
smp_mb__after_spinlock();

update_rq_clock(rq);

if (!preempt && prev_state) {
    try_to_block_task(rq, prev, &prev_state);
}

next = pick_next_task(rq, prev, &rf);

if (likely(prev != next)) {
    rq = context_switch(rq, prev, next, &rf);
}
```

#### `schedule()` - Public Scheduler Interface
```c
asmlinkage __visible void __sched schedule(void)
```

**User Interface**: Primary scheduling function called by kernel code

**Process**:
1. **Work Submission**: Handle worker thread notifications
2. **Scheduling Loop**: Call __schedule() until no reschedule needed
3. **Worker Update**: Update worker thread state after scheduling

### Task Selection and Picking

#### `pick_next_task()` - Next Task Selection
```c
static struct task_struct *pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
```

**Task Selection Strategy**:
1. **Core Scheduling**: Handle multi-threaded core coordination (if enabled)
2. **Fast Path**: Direct CFS selection for common case
3. **Class Iteration**: Try each scheduling class in priority order
4. **Fallback**: Always select idle task if no other tasks available

**Optimization for Common Case**:
```c
if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
           rq->nr_running == rq->cfs.h_nr_queued)) {
    p = pick_next_task_fair(rq, prev, rf);
    if (!p)
        p = pick_task_idle(rq);
    return p;
}
```

#### `__pick_next_task()` - Class-Based Selection
```c
static inline struct task_struct *__pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
```

**Class Priority Order**:
1. **Stop Class**: Migration and stop tasks
2. **Deadline Class**: Real-time deadline tasks
3. **RT Class**: Real-time FIFO/RR tasks
4. **Fair Class**: Normal time-sharing tasks (CFS)
5. **Idle Class**: Idle and very low priority tasks

### Context Switching

#### `context_switch()` - Complete Context Switch
```c
static __always_inline struct rq *context_switch(struct rq *rq, struct task_struct *prev, struct task_struct *next, struct rq_flags *rf)
```

**Context Switch Process**:
1. **Memory Management**: Switch virtual memory context
2. **Architecture Switch**: Switch CPU registers and stack
3. **Cleanup**: Handle post-switch cleanup and statistics
4. **Lock Management**: Proper lock handoff between tasks

**Key Components**:
- **switch_mm()**: Memory management context switch
- **switch_to()**: Architecture-specific register/stack switch
- **finish_task_switch()**: Post-switch cleanup and statistics

### Task State Management

#### `try_to_block_task()` - Handle Blocking Tasks
```c
static bool try_to_block_task(struct rq *rq, struct task_struct *p, unsigned long *task_state_p)
```

**Blocking Logic**:
1. **Signal Check**: Verify no pending signals that would prevent blocking
2. **Load Contribution**: Update load average contribution
3. **Special States**: Handle special task states appropriately
4. **Dequeue**: Remove task from run queue if blocking

#### Task State Transitions
```c
set_current_state(TASK_INTERRUPTIBLE);  /* Prepare to sleep */
schedule();                             /* Give up CPU */
set_current_state(TASK_RUNNING);        /* Back to running */
```

### Load Balancing and Migration

#### `balance_callback` - Load Balancing Framework
```c
struct balance_callback {
    struct balance_callback *next;
    void (*func)(struct rq *rq);
};
```

**Load Balancing Types**:
- **Periodic Balancing**: Regular load distribution
- **Idle Balancing**: Balance when CPU becomes idle
- **Newidle Balancing**: Balance before going idle
- **Active Balancing**: Force migration of specific tasks

#### SMP Scheduling Coordination
- **Cross-CPU Wake-ups**: Wake tasks on optimal CPUs
- **Migration**: Move tasks between CPUs for load balancing
- **Affinity**: Respect CPU affinity constraints
- **Topology**: Consider NUMA and cache topology

### Preemption and Tickless Operation

#### Preemption Models
- **Voluntary Preemption**: Tasks yield voluntarily
- **Preemptible Kernel**: Kernel can be preempted at most points
- **Real-Time Preemption**: Full real-time preemption support

#### `sched_tick()` - Periodic Scheduler Tick
```c
void sched_tick(void)
```

**Tick Processing**:
1. **Current Task Update**: Update current task statistics
2. **Preemption Check**: Check if preemption is needed
3. **Load Balancing**: Trigger periodic load balancing
4. **Accounting**: Update CPU time accounting

### Core Scheduling (Multi-Core Coordination)

#### Core Scheduling Overview
Core scheduling coordinates scheduling across SMT (Simultaneous Multi-Threading) cores to prevent unauthorized information leakage between security domains.

#### `pick_next_task()` with Core Scheduling
```c
#ifdef CONFIG_SCHED_CORE
static struct task_struct *pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
```

**Core-Wide Coordination**:
1. **Cookie Matching**: Ensure compatible security contexts
2. **SMT Coordination**: Coordinate across sibling threads
3. **Force Idle**: Force idle threads when no compatible tasks
4. **Security Isolation**: Maintain security boundaries

#### Core Scheduling States
- **Core Leader**: One CPU manages core-wide decisions
- **Core Followers**: Other CPUs follow leader's decisions
- **Force Idle**: Threads forced idle for security
- **Cookie Matching**: Security context compatibility

## Advanced Features

### Bandwidth Control and Throttling

#### RT Bandwidth Control
```c
struct rt_bandwidth {
    raw_spinlock_t      rt_runtime_lock;   /* Runtime lock */
    ktime_t             rt_period;         /* Period length */
    u64                 rt_runtime;        /* Runtime per period */
    struct hrtimer      rt_period_timer;   /* Period timer */
    unsigned int        rt_period_active;  /* Period active flag */
};
```

**Bandwidth Enforcement**:
- **Period Tracking**: Track bandwidth usage over time periods
- **Runtime Limits**: Enforce maximum runtime per period
- **Throttling**: Suspend tasks exceeding bandwidth limits
- **Inheritance**: Handle bandwidth inheritance in hierarchies

### CPU Hotplug Integration

#### `sched_cpu_starting()` - CPU Online
```c
int sched_cpu_starting(unsigned int cpu)
```

**CPU Activation Process**:
1. **Run Queue Setup**: Initialize per-CPU run queue
2. **Domain Assignment**: Assign to scheduling domains
3. **Load Balancing**: Enable load balancing for CPU
4. **Calibration**: Calibrate scheduling parameters

#### `sched_cpu_dying()` - CPU Offline
```c
int sched_cpu_dying(unsigned int cpu)
```

**CPU Deactivation Process**:
1. **Task Migration**: Migrate all tasks to other CPUs
2. **Load Balancing**: Disable load balancing
3. **Run Queue Cleanup**: Clean up per-CPU structures
4. **Domain Update**: Update scheduling domains

### Real-Time Scheduling Support

#### Deadline Scheduling
```c
struct sched_dl_entity {
    struct rb_node      rb_node;        /* RB-tree node */
    u64                 dl_runtime;     /* Remaining runtime */
    u64                 dl_deadline;    /* Absolute deadline */
    u64                 dl_period;      /* Period length */
    u64                 dl_bw;          /* Bandwidth */
    s64                 runtime;        /* Remaining runtime */
    u64                 deadline;       /* Current deadline */
    unsigned int        flags;          /* Scheduling flags */
    unsigned int        dl_throttled : 1;    /* Throttled flag */
    unsigned int        dl_yielded : 1;      /* Yielded flag */
    unsigned int        dl_non_contending : 1; /* Non-contending flag */
    unsigned int        dl_overrun : 1;      /* Overrun flag */
    struct hrtimer      dl_timer;       /* Deadline timer */
    struct hrtimer      inactive_timer; /* Inactive timer */
};
```

**Deadline Scheduling Features**:
- **EDF (Earliest Deadline First)**: Core scheduling algorithm
- **CBS (Constant Bandwidth Server)**: Bandwidth enforcement
- **Resource Reclaiming**: Reclaim unused bandwidth
- **Admission Control**: Verify schedulability before admission

### Performance Monitoring and Statistics

#### Scheduler Statistics
```c
struct sched_info {
    unsigned long       pcount;         /* Number of times scheduled */
    unsigned long long  run_delay;     /* Time spent waiting on run queue */
    unsigned long long  last_arrival;  /* Last time queued */
    unsigned long long  last_queued;   /* Last time run */
};
```

**Performance Metrics**:
- **Context Switch Rates**: Track switching frequency
- **Load Averages**: System load over time
- **CPU Utilization**: Per-CPU and per-task utilization
- **Cache Performance**: Cache miss rates and locality
- **Migration Costs**: Cost of task migration

### Power Management Integration

#### CPU Frequency Scaling
```c
struct sched_avg {
    u64                 last_update_time; /* Last update time */
    u64                 load_sum;        /* Load sum */
    u64                 runnable_sum;    /* Runnable sum */
    u32                 util_sum;        /* Utilization sum */
    u32                 period_contrib;  /* Period contribution */
    unsigned long       load_avg;       /* Load average */
    unsigned long       runnable_avg;   /* Runnable average */
    unsigned long       util_avg;       /* Utilization average */
    struct util_est     util_est;       /* Estimated utilization */
};
```

**Power-Aware Scheduling**:
- **DVFS Integration**: Dynamic voltage/frequency scaling
- **Idle States**: Coordinate with CPU idle states
- **Thermal Management**: Respond to thermal constraints
- **Energy Models**: Use energy models for task placement

## Error Handling and Recovery

### Scheduler Consistency Checks
- **Run Queue Validation**: Verify run queue consistency
- **Task State Validation**: Check task state transitions
- **Load Balance Verification**: Verify load balancing correctness
- **Deadlock Detection**: Detect potential scheduling deadlocks

### Recovery Mechanisms
- **Stuck Task Detection**: Detect and handle stuck tasks
- **Load Balance Recovery**: Recover from load balancing failures
- **Migration Failure Handling**: Handle task migration failures
- **Resource Leak Prevention**: Prevent scheduler resource leaks

### Debugging and Tracing
```c
trace_sched_switch(preempt, prev, next, prev_state);
trace_sched_wakeup(p);
trace_sched_migrate_task(p, new_cpu);
```

**Debugging Features**:
- **Scheduler Tracing**: Comprehensive trace events
- **Statistics Collection**: Detailed performance statistics
- **Debug Assertions**: Runtime consistency checking
- **Profiling Integration**: Integration with profiling tools

## Performance Optimizations

### Fast Path Optimizations
- **CFS Fast Path**: Optimized path for common CFS operations
- **Cache-Friendly Data Structures**: Optimize for CPU cache performance
- **Branch Prediction**: Optimize for common case branches
- **Lock Optimization**: Minimize lock contention and overhead

### SMP Scalability
- **Per-CPU Data Structures**: Minimize sharing between CPUs
- **Lock-Free Algorithms**: Use lock-free data structures where possible
- **NUMA Awareness**: Optimize for NUMA topology
- **Load Balancing Efficiency**: Efficient cross-CPU coordination

### Memory Access Optimization
- **Data Layout**: Optimize data structure layout for cache performance
- **Prefetching**: Strategic memory prefetching
- **False Sharing Avoidance**: Prevent false sharing between CPUs
- **Memory Barriers**: Minimal but correct memory ordering

## Integration Points

### Kernel Subsystem Integration
- **Memory Management**: Coordinate with MM for swap and memory pressure
- **I/O Subsystem**: Handle I/O wait and block device scheduling
- **Network Stack**: Coordinate with network packet processing
- **File Systems**: Handle file system operation scheduling

### User Space Interface
- **System Calls**: sched_setscheduler(), sched_yield(), etc.
- **Proc Files**: /proc/schedstat, /proc/sched_debug
- **CGroups**: Control group integration for resource limits
- **CPU Sets**: CPU affinity and isolation control

### Hardware Integration
- **Architecture Support**: Per-architecture context switching
- **SMP Support**: Multi-processor coordination
- **NUMA Support**: Non-uniform memory access optimization
- **Power Management**: CPU frequency and idle state coordination

This comprehensive scheduler implementation provides the foundation for fair, efficient, and responsive task scheduling in Linux, balancing performance, latency, and fairness requirements while supporting diverse workloads from real-time applications to batch processing jobs.