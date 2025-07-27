# kernel/sched/fair.c - Linux Completely Fair Scheduler (CFS)

## Overview

This file implements the Completely Fair Scheduler (CFS), the default process scheduler for normal tasks in Linux (SCHED_NORMAL/SCHED_BATCH). Originally developed by Ingo Molnar at Red Hat in 2007, CFS represents a paradigm shift from the previous O(1) scheduler, providing true proportional fairness through sophisticated virtual runtime accounting and red-black tree organization. The current implementation features the EEVDF (Earliest Eligible Virtual Deadline First) algorithm, advanced load balancing, hierarchical group scheduling, and extensive NUMA awareness.

## Historical Development

### Key Contributors and Evolution
- **Ingo Molnar (2007)**: Original CFS design and implementation
- **Mike Galbraith**: Interactivity improvements and wakeup preemption
- **Dmitry Adamushko**: Various algorithmic enhancements
- **Srivatsa Vaddagiri (IBM)**: Group scheduling and cgroups integration
- **Thomas Gleixner**: Scaled math optimizations
- **Peter Zijlstra**: Adaptive granularity and PELT (Per-Entity Load Tracking)
- **Modern Era**: EEVDF integration, Energy-Aware Scheduling (EAS), NUMA enhancements

### Evolution Timeline
- **2007**: Initial CFS implementation replacing O(1) scheduler
- **2008-2010**: Group scheduling and cgroups integration
- **2012-2015**: PELT introduction for accurate load tracking
- **2016-2018**: Energy-aware scheduling and NUMA balancing
- **2020-2023**: EEVDF algorithm integration and bandwidth control improvements

### Design Philosophy
CFS embodies the principle of complete fairness through proportional-share scheduling, where each task receives CPU time proportional to its weight. The design emphasizes low-latency interactive performance while maintaining fairness across diverse workloads, scalability to large SMP systems, and integration with modern hardware features.

## Core Architecture

### Scheduling Algorithm Evolution
```
Traditional Round Robin → O(1) Scheduler → CFS (Virtual Runtime) → EEVDF (Deadline-based)
      ↓                      ↓                  ↓                       ↓
[Fixed Timeslices]    [Priority Arrays]  [Proportional Share]   [Earliest Eligible Virtual Deadline First]
```

### Virtual Runtime Model
```
Physical Time → Weight-based Scaling → Virtual Runtime → Fairness Enforcement
      ↓               ↓                     ↓                ↓
[Real CPU Time]  [nice/cgroup weights]  [vruntime]     [Red-Black Tree Ordering]
```

### EEVDF Algorithm Components
1. **Eligibility**: Tasks must be owed service before selection
2. **Virtual Deadline**: Among eligible tasks, pick earliest deadline
3. **Slice Protection**: Tasks run until fair share achieved
4. **Preemption Control**: Balance responsiveness with fairness

## Key Data Structures

### Core Scheduling Entity
```c
struct sched_entity {
    struct load_weight      load;          /* Entity weight and priority */
    struct rb_node         run_node;      /* Red-black tree node */
    u64                    vruntime;      /* Virtual runtime */
    u64                    deadline;      /* EEVDF virtual deadline */
    u64                    min_vruntime;  /* Augmented tree: subtree minimum */
    u64                    min_slice;     /* Augmented tree: minimum slice */
    u64                    slice;         /* Current time slice */
    u64                    lag;           /* Lag from ideal fair share */
    struct sched_entity   *parent;       /* Group hierarchy parent */
    struct cfs_rq         *cfs_rq;       /* Runqueue this entity belongs to */
    struct cfs_rq         *my_q;         /* Runqueue owned by this entity */
    struct sched_avg       avg;          /* Per-Entity Load Tracking (PELT) */
    /* ... */
};
```

### CFS Runqueue Structure
```c
struct cfs_rq {
    struct load_weight      load;              /* Aggregate load of entities */
    unsigned int           nr_running;        /* Number of runnable entities */
    unsigned int           h_nr_running;      /* Hierarchical count */
    
    /* Virtual runtime tracking */
    u64                    min_vruntime;      /* Monotonic vruntime progress */
    u64                    avg_vruntime;      /* Weighted average vruntime */
    u64                    avg_load;          /* Total weight for average */
    
    /* Red-black tree for entity organization */
    struct rb_root_cached  tasks_timeline;   /* Cached leftmost optimization */
    
    /* Load tracking and statistics */
    struct sched_avg       avg;              /* Runqueue load averages */
    
    /* Group scheduling support */
    struct task_group      *tg;              /* Task group owner */
    struct sched_entity    *tg_se;           /* Group's scheduling entity */
    
    /* Bandwidth control */
    int                    throttled;        /* Throttling state */
    u64                    throttled_clock;  /* Time spent throttled */
    
    /* NUMA awareness */
    struct list_head       leaf_cfs_rq_list; /* Load balancing list */
    /* ... */
};
```

### Load Tracking (PELT) Structure
```c
struct sched_avg {
    u64                    last_update_time; /* Last PELT update */
    u64                    load_sum;         /* Accumulated load */
    u64                    runnable_sum;     /* Accumulated runnable time */
    u32                    util_sum;         /* Accumulated utilization */
    unsigned long          load_avg;        /* Exponentially decayed load */
    unsigned long          runnable_avg;    /* Exponentially decayed runnable */
    unsigned long          util_avg;        /* Exponentially decayed utilization */
    struct util_est        util_est;        /* Utilization estimation */
    /* ... */
};
```

### Group Scheduling Infrastructure
```c
struct task_group {
    struct sched_entity   **se;             /* Per-CPU scheduling entities */
    struct cfs_rq        **cfs_rq;          /* Per-CPU CFS runqueues */
    unsigned long          shares;          /* Proportional share weight */
    
    struct cfs_bandwidth   cfs_bandwidth;   /* Bandwidth control */
    
    struct task_group     *parent;          /* Hierarchy parent */
    struct list_head       children;        /* Child groups */
    struct list_head       siblings;        /* Sibling groups */
    /* ... */
};
```

## Core Functions

### Virtual Runtime Management

#### `entity_key()` - Virtual Runtime Calculation
```c
static u64 entity_key(struct cfs_rq *cfs_rq, struct sched_entity *se)
```

**Purpose**: Calculate relative virtual runtime for tree ordering

**Algorithm**:
```c
static u64 entity_key(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    return (s64)(se->vruntime - cfs_rq->min_vruntime);
}
```

**Key Properties**:
- **Overflow protection**: Relative calculation prevents 64-bit overflow
- **Monotonic ordering**: Ensures consistent tree ordering
- **Fairness metric**: Lower vruntime = higher scheduling priority

#### Virtual Time Advancement
```c
/* Core fairness equation: lag_i = S - s_i = w_i * (V - v_i) */
/* Where V = (Σ v_i * w_i) / W is the weighted average vruntime */

static void avg_vruntime_add(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    unsigned long weight = scale_load_down(se->load.weight);
    s64 key = entity_key(cfs_rq, se);
    
    cfs_rq->avg_vruntime += key * weight;
    cfs_rq->avg_load += weight;
}
```

### EEVDF Scheduling Algorithm

#### `pick_eevdf()` - Core Task Selection
```c
static struct sched_entity *pick_eevdf(struct cfs_rq *cfs_rq)
```

**Purpose**: Select next task using Earliest Eligible Virtual Deadline First algorithm

**Algorithm Flow**:
1. **Single entity optimization**: If only one task, return it
2. **Current task eligibility**: Check if current task remains eligible
3. **Leftmost eligible**: Try leftmost (lowest vruntime) task first
4. **Tree traversal**: Use augmented RB-tree for O(log n) eligible search

**Eligibility Check**:
```c
static bool entity_eligible(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    struct sched_entity *curr = cfs_rq->curr;
    s64 avg = cfs_rq->avg_vruntime;
    long load = cfs_rq->avg_load;
    
    if (curr && curr->on_rq) {
        unsigned long weight = scale_load_down(curr->load.weight);
        avg += entity_key(cfs_rq, curr) * weight;
        load += weight;
    }
    
    return avg >= (s64)(se->vruntime - se->lag) * load;
}
```

#### `enqueue_task_fair()` - Task Addition
```c
void enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
```

**Hierarchical Process**:
1. **Utilization estimation**: Update frequency scaling hints
2. **Group traversal**: Process each level of scheduling hierarchy
3. **Load tracking**: Update PELT averages for task and runqueue
4. **Slice assignment**: Set entity slice based on runqueue parameters
5. **Tree insertion**: Add to red-black tree with proper ordering

**Key Implementation**:
```c
for_each_sched_entity(se) {
    enqueue_entity(cfs_rq, se, flags);
    slice = cfs_rq_min_slice(cfs_rq);
    se->slice = slice;
    cfs_rq->h_nr_runnable += h_nr_runnable;
    
    if (cfs_rq_throttled(cfs_rq))
        break;
}
```

#### `dequeue_task_fair()` - Task Removal
```c
void dequeue_task_fair(struct rq *rq, struct task_struct *p, int flags)
```

**Cleanup Process**:
1. **Utilization tracking**: Update estimates for frequency scaling
2. **Hierarchical removal**: Remove from all hierarchy levels
3. **Load tracking**: Update PELT statistics
4. **Timer adjustment**: Update high-resolution tick if needed

### Preemption and Latency Control

#### `check_preempt_wakeup_fair()` - Wakeup Preemption
```c
static void check_preempt_wakeup_fair(struct rq *rq, struct task_struct *p, int wake_flags)
```

**Preemption Logic**:
1. **Policy checks**: Handle SCHED_BATCH and SCHED_IDLE restrictions
2. **EEVDF selection**: Use pick_eevdf() to determine if preemption needed
3. **Short slice preemption**: Allow quick tasks to preempt protected tasks
4. **Buddy system**: Respect cache locality hints

**Short Slice Optimization**:
```c
static inline bool do_preempt_short(struct cfs_rq *cfs_rq,
                                   struct sched_entity *pse, struct sched_entity *se)
{
    if (pse->slice >= se->slice) return false;  /* Shorter slice required */
    if (!entity_eligible(cfs_rq, pse)) return false;  /* Must be eligible */
    
    /* Additional EEVDF ordering checks for deadline preservation */
    return se->deadline > pse->deadline;
}
```

#### Slice Protection Mechanism
```c
static inline bool protect_slice(struct sched_entity *se)
{
    return (sched_feat(RUN_TO_PARITY) && se->vlag <= 0);
}
```

**Purpose**: Prevents excessive preemption of tasks before they achieve fairness

### Load Balancing Infrastructure

#### `sched_balance_rq()` - Core Load Balancing
```c
static int sched_balance_rq(int this_cpu, struct rq *this_rq,
                           struct sched_domain *sd, enum cpu_idle_type idle,
                           int *continue_balancing)
```

**Multi-Step Process**:
1. **Group analysis**: Find busiest scheduling domain group
2. **Runqueue selection**: Identify busiest runqueue within group
3. **Task migration**: Move tasks to balance load
4. **Imbalance resolution**: Handle various imbalance types

**Load Balancing Environment**:
```c
struct lb_env {
    struct sched_domain    *sd;
    struct rq              *src_rq;
    struct rq              *dst_rq;
    int                     src_cpu;
    int                     dst_cpu;
    enum cpu_idle_type      idle;
    long                    imbalance;
    struct cpumask         *cpus;
    unsigned int            flags;
    /* Migration strategy and constraints */
};
```

#### NUMA-Aware Load Balancing
```c
static bool migrate_degrades_locality(struct task_struct *p, struct lb_env *env)
```

**NUMA Considerations**:
- **Memory locality**: Prevent migrations that hurt NUMA performance
- **Node distances**: Consider topology when evaluating migrations
- **NUMA groups**: Respect shared memory access patterns
- **Preferred node**: Honor task's preferred NUMA node placement

### Group Scheduling and Bandwidth Control

#### Hierarchical Group Management
```c
#ifdef CONFIG_FAIR_GROUP_SCHED
#define for_each_sched_entity(se) \
    for (; se; se = se->parent)
```

**Group Scheduling Features**:
- **Proportional shares**: Each cgroup gets CPU time proportional to shares
- **Hierarchical fairness**: Fairness maintained across group hierarchy
- **Load aggregation**: Parent load = sum of child loads
- **Bandwidth control**: Hard limits via CFS_BANDWIDTH

#### Bandwidth Control Implementation
```c
struct cfs_bandwidth {
    raw_spinlock_t          lock;
    ktime_t                 period;           /* Bandwidth period (100ms default) */
    u64                     quota;            /* CPU quota per period */
    u64                     runtime;          /* Current runtime budget */
    u64                     burst;            /* Burst allowance */
    s64                     hierarchical_quota; /* Inherited quota limit */
    struct hrtimer          period_timer;    /* Period refresh timer */
    struct hrtimer          slack_timer;     /* Slack distribution timer */
    struct list_head        throttled_cfs_rq; /* Throttled runqueues */
    /* ... */
};
```

**Throttling Process**:
1. **Runtime tracking**: Monitor consumed CPU time per period
2. **Quota enforcement**: Throttle when quota exceeded
3. **Hierarchical throttling**: Propagate up group hierarchy
4. **Timer-based restoration**: Unthrottle when new period begins

### Per-Entity Load Tracking (PELT)

#### PELT Update Algorithm
```c
static void update_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
```

**Multi-Level Tracking**:
1. **Task level**: Individual entity load/utilization
2. **Group level**: Aggregated cfs_rq averages
3. **System level**: CPU-wide utilization

**Exponential Decay Model**:
- **Decay period**: ~1024ms (y^32 ≈ 0.5)
- **Update frequency**: Every scheduling event
- **Accuracy**: Millisecond-precision load tracking

#### Load Balancing Integration
```c
static void update_cfs_rq_load_avg(u64 now, struct cfs_rq *cfs_rq)
{
    struct sched_avg *sa = &cfs_rq->avg;
    int decayed = 0;
    
    if (cfs_rq->removed.nr) {
        /* Handle removed entities */
        unsigned long r = cfs_rq->removed.util_avg;
        sub_positive(&sa->util_avg, r);
        decayed = 1;
    }
    
    decayed |= __update_load_avg_cfs_rq(now, cfs_rq);
    return decayed;
}
```

### Energy-Aware Scheduling (EAS)

#### Energy-Efficient CPU Selection
```c
static int find_energy_efficient_cpu(struct task_struct *p, int prev_cpu, int sync_flag)
```

**Energy Model Integration**:
1. **Performance domains**: Group CPUs with similar energy characteristics
2. **Utilization-based placement**: Favor underutilized, energy-efficient cores
3. **Spare capacity**: Maximize cluster packing for energy savings
4. **Energy delta calculation**: Compare energy cost of different placements

**Energy Environment**:
```c
struct energy_env {
    unsigned long task_busy_time;     /* Task CPU requirement */
    unsigned long pd_busy_time;       /* Performance domain utilization */
    unsigned long cpu_cap;            /* CPU capacity */
    unsigned long pd_cap;             /* Performance domain capacity */
    /* ... */
};
```

## Advanced Features

### NUMA Balancing

#### NUMA Group Management
```c
struct numa_group {
    atomic_t               refcount;          /* Reference counting */
    spinlock_t             lock;              /* Group modification lock */
    int                    nr_tasks;          /* Number of tasks in group */
    pid_t                  gid;               /* Group identifier */
    int                    active_nodes;      /* Number of active NUMA nodes */
    struct rcu_head        rcu;               /* RCU cleanup */
    unsigned long          faults[];         /* Per-node fault statistics */
};
```

#### Memory-Aware Task Placement
```c
static bool should_numa_migrate_memory(struct task_struct *p, struct page *page,
                                     int src_nid, int dst_cpu)
```

**NUMA Migration Logic**:
- **Fault-driven placement**: Use memory access patterns for placement decisions
- **Group-aware migration**: Consider shared memory access in groups
- **Cost-benefit analysis**: Weigh migration cost against locality benefit
- **Topology awareness**: Respect NUMA distances and bandwidth

### Performance Optimizations

#### Cache-Friendly Design
```c
/* Cache line aligned structures */
struct uclamp_rq {
    unsigned int value;
    struct uclamp_bucket bucket[UCLAMP_BUCKETS];
} ____cacheline_aligned;

/* Per-CPU data to minimize contention */
DEFINE_PER_CPU(cpumask_var_t, load_balance_mask);
```

#### Lock-Free Algorithms
```c
/* RCU-protected domain traversal */
rcu_read_lock();
for_each_domain(cpu, sd) {
    /* Lock-free scheduling domain access */
    if (should_we_balance(&env)) {
        /* Minimal locking for balancing */
    }
}
rcu_read_unlock();
```

#### Fast Path Optimizations
- **Branch prediction**: Extensive use of `likely()/unlikely()` hints
- **Cached leftmost**: O(1) access to next schedulable task
- **Single task optimization**: Special case handling for minimal overhead
- **Hierarchical caching**: Avoid unnecessary group hierarchy traversal

## Integration Points

### Scheduler Class Integration
- **Class hierarchy**: RT, DL, CFS, IDLE priority ordering
- **Fair server**: CFS runs as deadline server under RT pressure
- **Bandwidth sharing**: Coordinated bandwidth allocation across classes

### Memory Management Integration
- **NUMA balancing**: Tight integration with memory management
- **Page fault handling**: Triggers NUMA migration decisions
- **Memory pressure**: Load balancing considers memory bandwidth

### Power Management Integration
- **CPU frequency scaling**: Utilization feedback for DVFS decisions
- **Thermal pressure**: Integration with thermal throttling
- **Energy awareness**: Power-efficient task placement

### Container and Virtualization Support
- **Cgroups integration**: Hierarchical resource control
- **Container isolation**: Bandwidth control for workload isolation
- **VM scheduling**: Coordination with hypervisor scheduling

This comprehensive CFS implementation represents the pinnacle of process scheduling engineering, balancing fairness, performance, scalability, and energy efficiency across diverse hardware platforms and workload characteristics while maintaining the responsiveness expected from modern interactive systems.