# Linux Kernel Scheduler Core (kernel/sched/core.c) - Comprehensive Documentation

This document provides an in-depth analysis of the Linux kernel's core scheduler implementation, focusing on the central scheduling mechanisms, load balancing, scheduling classes, real-time features, and power management integration.

## Table of Contents
1. [Overview and Architecture](#overview-and-architecture)
2. [Core Scheduling Loop and Task Switching](#core-scheduling-loop-and-task-switching)
3. [Load Balancing Across Multiple CPUs](#load-balancing-across-multiple-cpus)
4. [Scheduling Class Framework and Policy Implementation](#scheduling-class-framework-and-policy-implementation)
5. [Real-time Scheduling and Priority Management](#real-time-scheduling-and-priority-management)
6. [Power Management Integration and Energy Efficiency](#power-management-integration-and-energy-efficiency)
7. [SMP Coordination and CPU Topology Awareness](#smp-coordination-and-cpu-topology-awareness)
8. [Integration with Other Kernel Subsystems](#integration-with-other-kernel-subsystems)
9. [Key Data Structures](#key-data-structures)
10. [Performance Considerations and Optimizations](#performance-considerations-and-optimizations)

## Overview and Architecture

### System Architecture
The Linux kernel scheduler is a sophisticated multi-class scheduling system designed to handle diverse workload requirements across different computing scenarios - from embedded systems to large-scale servers.

**Key Design Principles:**
- **Modular scheduling classes**: Different policies for different task types (CFS, RT, DL, IDLE, EXT)
- **Per-CPU runqueues**: Scalable SMP design with minimal contention
- **Load balancing**: Dynamic work distribution across CPUs
- **Energy awareness**: Power-efficient scheduling decisions
- **Real-time support**: Deterministic scheduling for time-critical tasks

### Core Components Location
```
kernel/sched/
├── core.c           # Main scheduler logic (this file)
├── fair.c           # Completely Fair Scheduler (CFS)
├── rt.c             # Real-time scheduler (FIFO/RR)
├── deadline.c       # Deadline scheduler (EDF/CBS)
├── idle.c           # Idle task scheduler
├── ext.c            # Extended/BPF scheduler (sched_ext)
├── sched.h          # Internal scheduler definitions
├── topology.c       # CPU topology and domain management
└── ...
```

## Core Scheduling Loop and Task Switching

### Main Scheduling Function: `__schedule()`

The heart of the Linux scheduler is the `__schedule()` function (line 6662), which implements the core scheduling algorithm:

```c
static void __sched notrace __schedule(int sched_mode)
{
    struct task_struct *prev, *next;
    bool preempt = sched_mode > SM_NONE;
    bool is_switch = false;
    unsigned long *switch_count;
    unsigned long prev_state;
    struct rq_flags rf;
    struct rq *rq;
    
    // Main scheduling logic...
}
```

#### Scheduling Modes
The scheduler operates in different modes based on the context:
- `SM_IDLE (-1)`: Idle scheduling
- `SM_NONE (0)`: Normal voluntary scheduling
- `SM_PREEMPT (1)`: Preemptive scheduling
- `SM_RTLOCK_WAIT (2)`: RT lock waiting (PREEMPT_RT)

### Task Selection Algorithm: `__pick_next_task()`

The task selection process (line 6028) follows a hierarchical approach:

```c
static inline struct task_struct *
__pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    const struct sched_class *class;
    struct task_struct *p;
    
    // Fast path optimization for CFS-only scenarios
    if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
               rq->nr_running == rq->cfs.h_nr_queued)) {
        p = pick_next_task_fair(rq, prev, rf);
        if (unlikely(p == RETRY_TASK))
            goto restart;
        if (!p) {
            p = pick_task_idle(rq);
            put_prev_set_next_task(rq, prev, p);
        }
        return p;
    }
    
restart:
    prev_balance(rq, prev, rf);
    
    // Iterate through scheduling classes by priority
    for_each_active_class(class) {
        if (class->pick_next_task) {
            p = class->pick_next_task(rq, prev);
            if (p)
                return p;
        } else {
            p = class->pick_task(rq);
            if (p) {
                put_prev_set_next_task(rq, prev, p);
                return p;
            }
        }
    }
    
    BUG(); /* The idle class should always have a runnable task. */
}
```

#### Optimization: CFS Fast Path
The scheduler includes a significant optimization for workloads dominated by normal (CFS) tasks. When all runnable tasks belong to the fair scheduling class, the scheduler bypasses the full class iteration and directly calls the CFS picker.

### Context Switching: `context_switch()`

Context switching (line 5341) is one of the most performance-critical operations:

```c
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
               struct task_struct *next, struct rq_flags *rf)
{
    prepare_task_switch(rq, prev, next);
    
    arch_start_context_switch(prev);
    
    // Memory management context switching
    if (!next->mm) {                                // to kernel
        enter_lazy_tlb(prev->active_mm, next);
        next->active_mm = prev->active_mm;
        if (prev->mm)                           // from user
            mmgrab_lazy_tlb(prev->active_mm);
        else
            prev->active_mm = NULL;
    } else {                                        // to user
        membarrier_switch_mm(rq, prev->active_mm, next->mm);
        switch_mm_irqs_off(prev->active_mm, next->mm, next);
        lru_gen_use_mm(next->mm);
        
        if (!prev->mm) {                        // from kernel
            rq->prev_mm = prev->active_mm;
            prev->active_mm = NULL;
        }
    }
    
    // Architecture-specific register state switching
    switch_to(prev, next, prev);
    
    return finish_task_switch(prev);
}
```

#### Context Switch Optimizations
1. **Lazy TLB**: Kernel threads don't switch page tables unnecessarily
2. **Memory barrier optimizations**: Minimal barriers for performance
3. **MM context caching**: Reuse memory contexts when possible

## Load Balancing Across Multiple CPUs

### Load Balancing Architecture

The Linux scheduler implements sophisticated load balancing mechanisms to distribute work efficiently across multiple CPU cores. Load balancing operates at several levels:

#### Scheduling Domains Hierarchy
The scheduler organizes CPUs into hierarchical domains based on topology:
- **CPU level**: Individual CPU cores
- **SMT level**: Hyperthreading siblings
- **MC level**: Multi-core package
- **NUMA level**: NUMA nodes
- **System level**: Entire system

#### Load Balancing Triggers
1. **Periodic balancing**: Timer-driven load balancing
2. **Idle balancing**: When CPU becomes idle
3. **Wake-up balancing**: During task wake-up
4. **Fork balancing**: When new tasks are created

### Active Load Balancing

For heavily imbalanced scenarios, the scheduler employs active load balancing (line 12069):

```c
static int active_load_balance_cpu_stop(void *data)
{
    struct rq *busiest_rq = data;
    int busiest_cpu = cpu_of(busiest_rq);
    int target_cpu = busiest_rq->push_cpu;
    struct rq *target_rq = cpu_rq(target_cpu);
    struct sched_domain *sd;
    struct task_struct *p = NULL;
    struct rq_flags rf;
    
    rq_lock_irq(busiest_rq, &rf);
    
    // Find suitable task to migrate
    if (busiest_rq->nr_running <= 1)
        goto out_unlock;
        
    // Perform migration under appropriate domain
    for_each_domain(target_cpu, sd) {
        if (cpumask_test_cpu(busiest_cpu, sched_domain_span(sd)))
            break;
    }
    
    if (likely(sd)) {
        struct lb_env env = {
            .sd             = sd,
            .dst_cpu        = target_cpu,
            .dst_rq         = target_rq,
            .src_cpu        = busiest_cpu,
            .src_rq         = busiest_rq,
            .idle           = CPU_IDLE,
            .flags          = LBF_ACTIVE_LB,
        };
        
        // Attempt to find and migrate a task
        schedstat_inc(sd->alb_count);
        p = detach_one_task(&env);
        if (p) {
            schedstat_inc(sd->alb_pushed);
            // Migration will be completed by load balancer
        } else {
            schedstat_inc(sd->alb_failed);
        }
    }
    
out_unlock:
    busiest_rq->active_balance = 0;
    rq_unlock(busiest_rq, &rf);
    
    if (p)
        attach_one_task(target_rq, p);
        
    return 0;
}
```

### CPU Affinity and Migration Control

Tasks can control their CPU placement through:
- **CPU affinity masks**: `cpus_allowed` field in task_struct
- **Migration disable**: Temporary migration prevention
- **CPU isolation**: Exclude CPUs from load balancing

## Scheduling Class Framework and Policy Implementation

### Scheduling Class Hierarchy

The Linux scheduler supports multiple scheduling classes organized in priority order:

```c
// Scheduling class priority (highest to lowest)
extern const struct sched_class stop_sched_class;    // CPU stop tasks
extern const struct sched_class dl_sched_class;      // Deadline tasks
extern const struct sched_class rt_sched_class;      // Real-time tasks
extern const struct sched_class fair_sched_class;    // Normal tasks (CFS)
extern const struct sched_class idle_sched_class;    // Idle tasks
extern const struct sched_class ext_sched_class;     // Extended/BPF scheduler
```

### Scheduling Class Interface

Each scheduling class implements a standard interface (line 2390):

```c
struct sched_class {
#ifdef CONFIG_UCLAMP_TASK
    int uclamp_enabled;
#endif
    
    // Core scheduling operations
    void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
    bool (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
    void (*yield_task)   (struct rq *rq);
    bool (*yield_to_task)(struct rq *rq, struct task_struct *p);
    
    // Task selection
    void     (*wakeup_preempt)(struct rq *rq, struct task_struct *p, int flags);
    struct task_struct *(*pick_next_task)(struct rq *rq, struct task_struct *prev);
    struct task_struct *(*pick_task)(struct rq *rq);
    void (*put_prev_task)(struct rq *rq, struct task_struct *p, struct task_struct *next);
    void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);
    
    // Load balancing
#ifdef CONFIG_SMP
    int (*balance)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
    int (*select_task_rq)(struct task_struct *p, int task_cpu, int flags);
    void (*migrate_task_rq)(struct task_struct *p, int new_cpu);
    void (*task_woken)(struct rq *rq, struct task_struct *p);
    void (*set_cpus_allowed)(struct task_struct *p, struct affinity_context *ctx);
    void (*rq_online)(struct rq *rq);
    void (*rq_offline)(struct rq *rq);
    enum cpu_idle_type (*find_idlest_cpu)(struct sched_group *group,
                                         struct task_struct *p, int this_cpu);
#endif
    
    // Time management
    void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
    void (*task_fork)(struct task_struct *p);
    void (*task_dead)(struct task_struct *p);
    
    // State changes
    void (*switched_from)(struct rq *rq, struct task_struct *p);
    void (*switched_to)  (struct rq *rq, struct task_struct *p);
    void (*prio_changed) (struct rq *rq, struct task_struct *p, int oldprio);
    
    // Bandwidth control
    void (*update_curr)(struct rq *rq);
    
    // Statistics and debugging
    void (*task_change_group)(struct task_struct *p);
};
```

### Class Selection Logic

The scheduler iterates through classes in priority order:

```c
#define for_each_active_class(class)                    \
    for_active_class_range(class, __sched_class_highest, __sched_class_lowest)

static inline const struct sched_class *next_active_class(const struct sched_class *class)
{
    class++;
#ifdef CONFIG_SCHED_CLASS_EXT
    if (scx_switched_all() && class == &fair_sched_class)
        class++;
    if (!scx_enabled() && class == &ext_sched_class)
        class++;
#endif
    return class;
}
```

### Completely Fair Scheduler (CFS)

CFS is the default scheduler for normal tasks, implementing the following principles:
- **Virtual runtime tracking**: Tasks accumulate `vruntime` as they run
- **Red-black tree organization**: Tasks sorted by vruntime
- **Target latency**: Configurable scheduling period
- **Minimum granularity**: Prevents excessive preemption

### Real-time Scheduling Classes

#### SCHED_FIFO and SCHED_RR
- **FIFO**: First-in-first-out within priority levels
- **Round-robin**: Time-sliced execution within priority levels
- **Priority range**: 1-99 (99 being highest)
- **Bandwidth control**: RT throttling to prevent system lockup

#### SCHED_DEADLINE
- **Earliest Deadline First (EDF)**: Global deadline ordering
- **Constant Bandwidth Server (CBS)**: Resource reservation
- **Admission control**: Schedulability analysis
- **Deadline inheritance**: Priority inversion prevention

## Real-time Scheduling and Priority Management

### Real-time Bandwidth Control

The RT scheduler implements bandwidth throttling to prevent real-time tasks from monopolizing the system:

```c
// Default RT scheduling parameters
int sysctl_sched_rt_period = 1000000;    // 1 second
int sysctl_sched_rt_runtime = 950000;    // 0.95 seconds (95%)
```

### Priority Inheritance Protocol

For real-time tasks, the scheduler implements priority inheritance to prevent priority inversion:

1. **Mutex ownership tracking**: Track RT task mutex ownership
2. **Priority boosting**: Boost mutex owner to highest waiting task priority
3. **Chain inheritance**: Handle transitive priority inheritance
4. **Deadlock detection**: Prevent inheritance cycles

### Deadline Scheduling Implementation

The deadline scheduler provides guaranteed bandwidth for time-critical tasks:

#### Key Parameters
- **Runtime**: Maximum execution time per period
- **Deadline**: Task completion deadline
- **Period**: Task activation period

#### Bandwidth Validation
```c
// Admission control check
U = Σ(runtime_i / period_i) ≤ 1.0
```

### CPU Isolation for Real-time

Real-time workloads benefit from CPU isolation:
- **Isolcpus**: Boot parameter to isolate CPUs
- **No HZ full**: Eliminate timer interrupts on isolated CPUs
- **RCU nocb**: Offload RCU callbacks from isolated CPUs

## Power Management Integration and Energy Efficiency

### Energy Aware Scheduling (EAS)

The scheduler incorporates energy-aware decisions when supported:

```c
// Energy model integration
#include <linux/energy_model.h>
```

#### Energy Model Components
1. **Performance domains**: Groups of CPUs with shared performance states
2. **Capacity states**: Different CPU frequency/voltage operating points
3. **Energy costs**: Power consumption at each capacity state

### CPU Frequency Scaling Integration

The scheduler interfaces with CPUfreq governors:
- **Utilization signals**: Provide CPU utilization to frequency governors
- **Deadline constraints**: Ensure deadline tasks get sufficient performance
- **Energy efficiency**: Balance performance and power consumption

### Idle CPU Management

Idle CPU handling optimizations:
```c
static inline void update_idle_core(struct rq *rq)
{
    if (static_branch_unlikely(&sched_smt_present))
        __update_idle_core(rq);
}
```

#### Idle State Selection
- **CPUIdle integration**: Select appropriate C-states
- **Wake latency consideration**: Balance power savings vs responsiveness
- **Topology awareness**: Consider SMT and package-level idle states

## SMP Coordination and CPU Topology Awareness

### Per-CPU Runqueues

Each CPU maintains its own runqueue to minimize contention:

```c
DECLARE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);

#define cpu_rq(cpu)     (&per_cpu(runqueues, (cpu)))
#define this_rq()       this_cpu_ptr(&runqueues)
#define task_rq(p)      cpu_rq(task_cpu(p))
```

### Core Scheduling (CONFIG_SCHED_CORE)

For security-sensitive environments, core scheduling ensures tasks sharing CPU cores are from the same security domain:

```c
#ifdef CONFIG_SCHED_CORE
static inline bool sched_core_enabled(struct rq *rq)
{
    return static_branch_unlikely(&__sched_core_enabled) && rq->core_enabled;
}
```

#### Core Scheduling Features
- **Cookie matching**: Tasks with same security cookie can share cores
- **Forced idle**: Idle cores when no compatible tasks available
- **Core-wide selection**: Coordinated task selection across core siblings

### NUMA Awareness

The scheduler considers NUMA topology for:
- **Memory locality**: Prefer local memory access
- **Load balancing**: Balance load within NUMA nodes first
- **Task placement**: Consider memory placement during wake-up

### SMT (Simultaneous Multithreading) Support

For systems with hyperthreading:
- **SMT-aware load balancing**: Consider thread capacity
- **Core utilization**: Track both physical and logical CPU usage
- **Idle core detection**: Identify completely idle cores

## Integration with Other Kernel Subsystems

### Memory Management Integration

The scheduler integrates closely with the memory management subsystem:

#### Memory Context Switching
```c
// Memory management during context switch
if (!next->mm) {                                // to kernel
    enter_lazy_tlb(prev->active_mm, next);
    // ... lazy TLB handling
} else {                                        // to user
    switch_mm_irqs_off(prev->active_mm, next->mm, next);
    // ... full MM switch
}
```

#### Memory Barriers
Critical memory barriers ensure proper ordering:
- **smp_mb__after_spinlock()**: After acquiring runqueue lock
- **smp_store_release()**: When updating task state
- **smp_load_acquire()**: When reading shared state

### Interrupt and Softirq Integration

#### Preemption Control
```c
// Preemption points
void preempt_schedule(void);
void preempt_schedule_irq(void);
```

#### Timer Integration
- **Scheduler tick**: Regular timer interrupt for time accounting
- **High-resolution timers**: Precise deadline enforcement
- **No HZ**: Tickless operation for idle CPUs

### Tracing and Debugging

The scheduler provides extensive tracing support:
```c
#include <trace/events/sched.h>

// Trace points for debugging
trace_sched_switch(prev, next);
trace_sched_wakeup(p);
trace_sched_migrate_task(p, new_cpu);
```

### Cgroup Integration

Control groups allow resource limitation:
- **CPU bandwidth**: Limit CPU usage per cgroup
- **CPU shares**: Proportional CPU allocation
- **RT bandwidth**: Real-time CPU limits per cgroup

## Key Data Structures

### struct rq (Runqueue)

The central per-CPU data structure (line 1099):

```c
struct rq {
    raw_spinlock_t      __lock;         // Runqueue lock
    unsigned int        nr_running;     // Number of runnable tasks
    
    // Load tracking
    struct load_weight  load;
    unsigned long       nr_load_updates;
    u64                 nr_switches;
    
    // Scheduling classes runqueues
    struct cfs_rq       cfs;            // CFS runqueue
    struct rt_rq        rt;             // RT runqueue  
    struct dl_rq        dl;             // Deadline runqueue
    
    // Current task
    struct task_struct  *curr;          // Currently running task
    struct task_struct  *next;          // Next task to run
    struct task_struct  *stop;          // Stop machine task
    
    // Idle handling
    struct task_struct  *idle;          // Idle task
    
    // Clock and time tracking
    u64                 clock;          // Runqueue clock
    u64                 clock_task;     // Task clock
    u64                 clock_pelt;     // PELT clock
    unsigned int        clock_update_flags;
    
    // CPU information
    int                 cpu;            // CPU number
    int                 online;         // CPU online status
    
    // Load balancing
    struct list_head    leaf_cfs_rq_list;
    struct list_head    *tmp_alone_branch;
    struct balance_callback *balance_callback;
    
    // Statistics
    unsigned int        ttwu_count;     // Total wakeups
    unsigned int        ttwu_local;     // Local wakeups
    
#ifdef CONFIG_SMP
    struct root_domain  *rd;            // Root domain
    struct sched_domain *sd;            // Scheduling domain
    
    unsigned long       cpu_capacity;   // CPU capacity
    unsigned long       cpu_capacity_orig; // Original capacity
    
    struct callback_head *balance_callback;
    
    // Migration
    int                 push_cpu;       // CPU for active balancing
    struct cpu_stop_work active_balance_work;
    
    // NUMA balancing
    unsigned int        nr_numa_running;
    unsigned int        nr_preferred_running;
#endif

#ifdef CONFIG_SCHED_CORE
    // Core scheduling
    struct rq           *core;          // Core root
    struct task_struct  *core_pick;     // Core-wide pick
    unsigned int        core_enabled;   // Core scheduling enabled
    unsigned int        core_sched_seq; // Core scheduling sequence
    struct rb_root      core_tree;      // Core task tree
    unsigned long       core_cookie;    // Security cookie
#endif
};
```

### struct task_struct (Process Descriptor)

Key scheduling-related fields in the process descriptor:

```c
struct task_struct {
    volatile long       __state;        // Task state (RUNNING, SLEEPING, etc.)
    
    // Scheduling
    int                 prio;           // Dynamic priority
    int                 static_prio;    // Static priority  
    int                 normal_prio;    // Normal priority
    int                 rt_priority;    // Real-time priority
    
    const struct sched_class *sched_class;  // Scheduling class
    struct sched_entity se;             // CFS scheduling entity
    struct sched_rt_entity rt;          // RT scheduling entity
    struct sched_dl_entity dl;          // Deadline scheduling entity
    
    // CPU affinity
    struct cpumask      *cpus_ptr;      // Allowed CPUs
    cpumask_t           cpus_mask;      // CPU mask storage
    int                 nr_cpus_allowed; // Number of allowed CPUs
    
    // On runqueue state
    int                 on_rq;          // On runqueue flag
    int                 on_cpu;         // Currently running flag
    
    // Timing
    u64                 se.sum_exec_runtime;  // Total runtime
    u64                 se.vruntime;          // Virtual runtime (CFS)
    
    // CPU and NUMA
    int                 cpu;            // Current CPU
    int                 wake_cpu;       // CPU for wake-up
    
    // Memory management
    struct mm_struct    *mm;            // Memory descriptor
    struct mm_struct    *active_mm;     // Active memory descriptor
    
    // Core scheduling
#ifdef CONFIG_SCHED_CORE
    unsigned long       core_cookie;    // Core scheduling cookie
    struct rb_node      core_node;      // Core tree node
#endif
};
```

### struct sched_entity (CFS Entity)

CFS-specific scheduling information:

```c
struct sched_entity {
    struct load_weight  load;          // Load weight
    struct rb_node      run_node;      // Red-black tree node
    struct list_head    group_node;    // Group list node
    unsigned int        on_rq;         // On runqueue flag
    
    u64                 exec_start;    // Execution start time
    u64                 sum_exec_runtime; // Total execution time
    u64                 vruntime;      // Virtual runtime
    u64                 prev_sum_exec_runtime; // Previous runtime
    
    u64                 nr_migrations; // Number of migrations
    
    // Fair scheduling
    struct sched_statistics statistics;
    
#ifdef CONFIG_FAIR_GROUP_SCHED
    int                 depth;         // Hierarchy depth
    struct sched_entity *parent;      // Parent entity
    struct cfs_rq       *cfs_rq;      // CFS runqueue
    struct cfs_rq       *my_q;        // My group runqueue
#endif
    
#ifdef CONFIG_SMP
    struct sched_avg    avg;           // Load average tracking
#endif
};
```

## Performance Considerations and Optimizations

### Fast Path Optimizations

#### CFS-Only Fast Path
When all tasks belong to the fair scheduling class, the scheduler bypasses the full class iteration:

```c
if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
           rq->nr_running == rq->cfs.h_nr_queued)) {
    p = pick_next_task_fair(rq, prev, rf);
    // ... fast path execution
}
```

#### Cache-Friendly Data Layout
- **Per-CPU runqueues**: Minimize cache line bouncing
- **Hot/cold data separation**: Frequently accessed fields together
- **Alignment**: Proper cache line alignment for critical structures

### Scalability Optimizations

#### Lock Granularity
- **Per-runqueue locks**: Independent CPU scheduling
- **Lock-free operations**: Where possible, use atomic operations
- **Lock ordering**: Consistent ordering to prevent deadlocks

#### Load Balancing Efficiency
- **Hierarchical domains**: Limit balancing scope
- **Lazy balancing**: Defer balancing when unnecessary
- **Batch operations**: Group related operations

### Memory Efficiency

#### PELT (Per-Entity Load Tracking)
Efficient load tracking using geometric series:
```c
// Load tracking with decay
load_avg = load_avg * y^n + load * (1 - y^n)
```

#### Scheduling Statistics
Optional detailed statistics controlled by CONFIG_SCHEDSTATS.

### Real-time Optimizations

#### Priority Ceiling Protocol
- **Immediate priority inheritance**: Prevent priority inversion
- **Bounded blocking time**: Predictable worst-case behavior

#### CPU Isolation
- **Dedicated CPUs**: Reserve CPUs for RT tasks
- **Interrupt steering**: Route interrupts away from RT CPUs
- **NO_HZ_FULL**: Eliminate timer tick overhead

### Energy Efficiency

#### Dynamic Voltage and Frequency Scaling (DVFS)
- **Utilization-based scaling**: Scale frequency based on CPU usage
- **Deadline-aware scaling**: Ensure deadline constraints
- **Energy-aware task placement**: Consider energy costs in migration

#### Idle State Management
- **Aggressive C-states**: Use deep idle states when possible
- **Wake prediction**: Predict task wake-up patterns
- **Idle balancing**: Consolidate work to allow idle states

## Conclusion

The Linux kernel scheduler represents one of the most sophisticated and well-engineered components of modern operating systems. Its multi-class architecture, advanced load balancing, real-time capabilities, and energy awareness make it suitable for an incredibly diverse range of computing environments.

Key strengths include:
- **Scalability**: Efficient operation from embedded to large-scale systems
- **Fairness**: CFS provides excellent interactive performance
- **Real-time support**: Deterministic behavior for time-critical applications
- **Energy efficiency**: Power-aware scheduling decisions
- **Maintainability**: Clean, modular architecture

The scheduler continues to evolve with new features like sched_ext (eBPF scheduling), improved NUMA support, and enhanced energy awareness, ensuring it remains at the forefront of operating system design.

---

*This documentation covers the Linux kernel scheduler as of kernel version 6.16-rc7. The scheduler is actively developed, and implementation details may change in future versions.*