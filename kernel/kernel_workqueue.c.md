# kernel/workqueue.c - Linux Work Queue Implementation

## Overview

This file implements the Linux kernel's generic asynchronous execution mechanism with shared worker pools, originally developed by Ingo Molnar in 2002 and significantly redesigned by Tejun Heo in 2010. The workqueue subsystem provides a framework for deferring work execution to process context, enabling efficient asynchronous processing throughout the kernel. Work items are executed by kernel threads called workers, which are managed through shared worker pools that automatically scale based on workload demand.

## Historical Development

### Key Contributors and Evolution
- **Ingo Molnar (2002)**: Original workqueue implementation
- **David Woodhouse, Andrew Morton, Kai Petzke, Theodore Ts'o**: Earlier taskqueue/keventd foundation
- **Christoph Lameter**: alloc_percpu integration
- **Tejun Heo - SUSE Linux Products GmbH (2010)**: Major redesign with concurrent managed worker pools
- **Community Contributors**: Ongoing enhancements and optimizations

### Design Evolution
- **Pre-2010**: Simple per-CPU worker threads with limited concurrency
- **2010 Redesign**: Shared worker pools with automatic concurrency management
- **Modern Era**: Enhanced NUMA awareness, CPU isolation support, and container integration

### Design Philosophy
The workqueue system is built around the principle of shared, automatically managed worker pools that provide optimal concurrency while preventing resource exhaustion and maintaining system responsiveness.

## Core Concepts

### Work Queue Architecture

#### System Overview
```
Work Items → Work Queues → Pool Work Queues → Worker Pools → Worker Threads
     ↓           ↓              ↓               ↓              ↓
[Functions]  [Queuing]    [Per-CPU/Node]   [Thread Mgmt]  [Execution]
```

#### Worker Pool Types
- **Per-CPU Pools**: Two pools per CPU (normal and high priority)
- **Unbound Pools**: Dynamic pools not tied to specific CPUs
- **BH Pools**: Bottom-half execution pools for interrupt context
- **Ordered Pools**: Single-threaded execution for ordering guarantees

#### Concurrency Management
- **Automatic Scaling**: Dynamic worker creation and destruction
- **CPU Intensive Detection**: Automatic detection and isolation of long-running work
- **Congestion Management**: Prevents worker pool starvation
- **NUMA Awareness**: Optimized worker placement across NUMA nodes

## Key Data Structures

### `struct worker_pool` - Core Worker Pool
```c
struct worker_pool {
    raw_spinlock_t      lock;           /* Pool protection */
    int                 cpu;            /* Associated CPU */
    int                 node;           /* NUMA node ID */
    int                 id;             /* Pool identifier */
    unsigned int        flags;          /* Pool state flags */
    
    unsigned long       watchdog_ts;    /* Watchdog timestamp */
    bool                cpu_stall;      /* CPU stall detection */
    int                 nr_running;     /* Running worker count */
    
    struct list_head    worklist;       /* Pending work items */
    int                 nr_workers;     /* Total worker count */
    int                 nr_idle;        /* Idle worker count */
    
    struct list_head    idle_list;      /* Idle worker list */
    struct timer_list   idle_timer;     /* Worker timeout timer */
    struct work_struct  idle_cull_work; /* Worker cleanup work */
    struct timer_list   mayday_timer;   /* Emergency worker timer */
    
    DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);
                                        /* Busy worker hash table */
    struct worker      *manager;        /* Pool manager worker */
    struct list_head   workers;         /* All workers list */
    struct ida         worker_ida;      /* Worker ID allocator */
    
    struct workqueue_attrs *attrs;      /* Worker attributes */
    struct hlist_node  hash_node;       /* Unbound pool hash */
    int                refcnt;          /* Reference count */
    struct rcu_head    rcu;             /* RCU destruction */
};
```

### `struct workqueue_struct` - Work Queue Descriptor
```c
struct workqueue_struct {
    struct list_head    pwqs;           /* Pool workqueues */
    struct list_head    list;           /* Global workqueue list */
    
    struct mutex        mutex;          /* Workqueue protection */
    int                 work_color;     /* Current work color */
    int                 flush_color;    /* Flush operation color */
    atomic_t            nr_pwqs_to_flush; /* Flush coordination */
    
    struct wq_flusher  *first_flusher;  /* Flush wait queue */
    struct list_head    flusher_queue;  /* Flush waiters */
    struct list_head    flusher_overflow; /* Overflow flushers */
    
    struct list_head    maydays;        /* Emergency rescue requests */
    struct worker      *rescuer;        /* Rescue worker thread */
    int                 nr_drainers;    /* Drain operation count */
    
    int                 max_active;     /* Maximum active work items */
    int                 min_active;     /* Minimum active work items */
    int                 saved_max_active; /* Saved during freeze */
    int                 saved_min_active; /* Saved during freeze */
    
    struct workqueue_attrs *unbound_attrs; /* Unbound attributes */
    struct pool_workqueue __rcu *dfl_pwq; /* Default pwq for unbound */
    
    char                name[WQ_NAME_LEN]; /* Workqueue name */
    unsigned int        flags;          /* Workqueue flags */
    struct pool_workqueue __rcu * __percpu *cpu_pwq; /* Per-CPU pwqs */
    struct wq_node_nr_active *node_nr_active[]; /* Per-node active counts */
};
```

### `struct pool_workqueue` - Pool-Workqueue Association
```c
struct pool_workqueue {
    struct worker_pool *pool;           /* Associated worker pool */
    struct workqueue_struct *wq;       /* Owning workqueue */
    int                work_color;      /* Current work color */
    int                flush_color;     /* Flush color */
    int                refcnt;          /* Reference count */
    int                nr_in_flight[WORK_NR_COLORS]; /* In-flight work counts */
    bool               plugged;         /* Execution suspended */
    
    int                nr_active;       /* Active work count */
    struct list_head   inactive_works;  /* Inactive work queue */
    struct list_head   pending_node;    /* NUMA pending list */
    struct list_head   pwqs_node;       /* Workqueue pwq list */
    struct list_head   mayday_node;     /* Emergency rescue list */
    
    u64                stats[PWQ_NR_STATS]; /* Performance statistics */
    struct kthread_work release_work;   /* Async release work */
    struct rcu_head    rcu;             /* RCU protection */
};
```

### Worker State Management
```c
enum worker_flags {
    WORKER_DIE              = 1 << 1,   /* Worker termination */
    WORKER_IDLE             = 1 << 2,   /* Worker idle state */
    WORKER_PREP             = 1 << 3,   /* Preparing to work */
    WORKER_CPU_INTENSIVE    = 1 << 6,   /* CPU intensive work */
    WORKER_UNBOUND          = 1 << 7,   /* Unbound worker */
    WORKER_REBOUND          = 1 << 8,   /* Worker rebound */
    
    WORKER_NOT_RUNNING = WORKER_PREP | WORKER_CPU_INTENSIVE |
                        WORKER_UNBOUND | WORKER_REBOUND,
};

enum worker_pool_flags {
    POOL_BH             = 1 << 0,       /* Bottom-half pool */
    POOL_MANAGER_ACTIVE = 1 << 1,       /* Manager active */
    POOL_DISASSOCIATED  = 1 << 2,       /* CPU disassociated */
    POOL_BH_DRAINING    = 1 << 3,       /* BH pool draining */
};
```

## Core Functions

### Work Queue Creation and Management

#### `alloc_workqueue()` - Create Work Queue
**Purpose**: Allocate and initialize a new workqueue with specified attributes

**Key Features**:
- **Flexible Configuration**: Support for various workqueue types and attributes
- **NUMA Optimization**: Automatic NUMA-aware worker pool selection
- **Resource Management**: Proper resource allocation and error handling
- **Attribute Validation**: Comprehensive attribute validation and defaults

#### Work Queue Flags and Attributes
```c
/* Workqueue flags */
#define WQ_UNBOUND          0x0002      /* Not bound to any CPU */
#define WQ_FREEZABLE        0x0004      /* Participate in suspend freeze */
#define WQ_MEM_RECLAIM      0x0008      /* May be used for memory reclaim */
#define WQ_HIGHPRI          0x0010      /* High priority work */
#define WQ_CPU_INTENSIVE    0x0020      /* CPU intensive work */
#define WQ_SYSFS            0x0040      /* Visible in sysfs */
#define WQ_POWER_EFFICIENT  0x0080      /* Power efficient execution */
```

### Work Item Queuing and Execution

#### `__queue_work()` - Core Work Queuing Implementation
```c
static void __queue_work(int cpu, struct workqueue_struct *wq,
                        struct work_struct *work)
```

**Queuing Process**:
1. **Target Selection**: Determine target CPU and worker pool
2. **Conflict Detection**: Check for currently executing work conflicts
3. **Pool Selection**: Select appropriate worker pool for execution
4. **Concurrency Control**: Apply max_active limits and queuing logic
5. **Work Insertion**: Insert work into appropriate queue
6. **Worker Notification**: Wake workers if needed

**Key Optimizations**:
```c
/* Fast path for CPU selection */
if (req_cpu == WORK_CPU_UNBOUND) {
    if (wq->flags & WQ_UNBOUND)
        cpu = wq_select_unbound_cpu(raw_smp_processor_id());
    else
        cpu = raw_smp_processor_id();
}

/* Concurrency management */
if (list_empty(&pwq->inactive_works) && pwq_tryinc_nr_active(pwq, false)) {
    trace_workqueue_activate_work(work);
    insert_work(pwq, work, &pool->worklist, work_flags);
    kick_pool(pool);
} else {
    work_flags |= WORK_STRUCT_INACTIVE;
    insert_work(pwq, work, &pwq->inactive_works, work_flags);
}
```

#### `queue_work_on()` - CPU-Specific Work Queuing
```c
bool queue_work_on(int cpu, struct workqueue_struct *wq,
                  struct work_struct *work)
```

**Features**:
- **CPU Affinity**: Execute work on specific CPU
- **Validation**: Ensure target CPU is valid and online
- **Fallback Handling**: Graceful handling of offline CPUs
- **Return Value**: Indicates if work was successfully queued

#### `queue_work_node()` - NUMA-Aware Work Queuing
```c
bool queue_work_node(int node, struct workqueue_struct *wq,
                    struct work_struct *work)
```

**NUMA Optimizations**:
- **Node Locality**: Prefer execution on specified NUMA node
- **CPU Selection**: Select optimal CPU within node
- **Fallback Strategy**: Graceful degradation to any available CPU
- **Performance**: Minimize inter-node memory access

### Worker Thread Management

#### `worker_thread()` - Main Worker Thread Function
```c
static int worker_thread(void *__worker)
```

**Worker Lifecycle**:
1. **Initialization**: Set worker flags and enter main loop
2. **Work Processing**: Continuously process available work items
3. **Concurrency Management**: Participate in pool concurrency control
4. **Idle Management**: Enter idle state when no work available
5. **Termination**: Clean exit when worker is no longer needed

**Main Processing Loop**:
```c
do {
    struct work_struct *work = list_first_entry(&pool->worklist,
                                               struct work_struct, entry);
    if (assign_work(work, worker, NULL))
        process_scheduled_works(worker);
} while (keep_working(pool));
```

#### `process_scheduled_works()` - Work Execution Engine
**Purpose**: Execute assigned work items with proper context management

**Execution Context**:
- **Function Invocation**: Call work function with proper setup
- **Exception Handling**: Handle work function exceptions gracefully
- **CPU Intensive Detection**: Monitor execution time and adjust flags
- **Statistics Tracking**: Update performance and execution statistics

#### Worker Pool Management

#### `create_worker()` - Worker Creation
**Process**:
1. **Worker Allocation**: Allocate worker structure and assign ID
2. **Thread Creation**: Create kernel thread with appropriate attributes
3. **Pool Association**: Associate worker with target pool
4. **Initialization**: Initialize worker state and statistics
5. **Activation**: Start worker thread execution

#### `destroy_worker()` - Worker Termination
**Process**:
1. **Termination Signal**: Set WORKER_DIE flag
2. **Graceful Shutdown**: Allow current work completion
3. **Resource Cleanup**: Clean up worker resources and statistics
4. **Pool Update**: Update pool worker counts and state

### Concurrency Management

#### `manage_workers()` - Dynamic Worker Management
**Purpose**: Automatically adjust worker pool size based on workload

**Management Logic**:
- **Need Assessment**: Determine if more workers are needed
- **Creation Logic**: Create new workers under appropriate conditions
- **Destruction Logic**: Remove idle workers when excess capacity exists
- **Resource Limits**: Respect system resource limits and constraints

#### CPU Intensive Work Handling
```c
/* CPU intensive threshold detection */
static unsigned long wq_cpu_intensive_thresh_us = ULONG_MAX;

/* Mark worker as CPU intensive */
if (worker_set_flags(worker, WORKER_CPU_INTENSIVE))
    pwq->stats[PWQ_STAT_CPU_INTENSIVE]++;
```

**CPU Intensive Features**:
- **Automatic Detection**: Monitor work execution time
- **Concurrency Exclusion**: Exclude from normal concurrency management
- **Resource Protection**: Prevent CPU intensive work from blocking others
- **Statistics Tracking**: Track CPU intensive violations

### Rescue Operations

#### `rescuer_thread()` - Emergency Worker Management
**Purpose**: Provide guaranteed work execution during memory pressure

**Rescue Scenarios**:
- **Memory Allocation Failures**: When worker creation fails due to memory pressure
- **Pool Starvation**: When all workers are blocked waiting for memory
- **Deadlock Prevention**: Prevent memory reclaim deadlocks
- **Critical Work Execution**: Ensure critical work can always execute

**Rescue Process**:
1. **Mayday Detection**: Detect when pools need rescue assistance
2. **Work Identification**: Identify work items requiring rescue
3. **Direct Execution**: Execute work items directly in rescuer context
4. **Resource Conservation**: Minimize resource usage during rescue

### Work Queue Flushing and Synchronization

#### `flush_workqueue()` - Workqueue Synchronization
**Purpose**: Wait for all currently queued work items to complete

**Flush Mechanism**:
1. **Color Management**: Use color-based flush coordination
2. **Completion Tracking**: Track work completion across all pools
3. **Barrier Insertion**: Insert flush barriers to establish completion points
4. **Wait Coordination**: Coordinate multiple concurrent flush operations

#### `drain_workqueue()` - Workqueue Draining
**Purpose**: Prevent new work queuing and wait for completion

**Drain Process**:
1. **Queue Blocking**: Block new work item queuing
2. **Current Work Completion**: Wait for all current work to complete
3. **Resource Cleanup**: Clean up workqueue resources
4. **State Restoration**: Restore workqueue to operational state

## Advanced Features

### NUMA and CPU Affinity Management

#### `struct wq_node_nr_active` - Per-Node Active Work Tracking
```c
struct wq_node_nr_active {
    int            max;                 /* Per-node max_active */
    atomic_t       nr;                  /* Per-node nr_active */
    raw_spinlock_t lock;               /* Protection lock */
    struct list_head pending_pwqs;     /* Pending pwqs */
};
```

**NUMA Optimizations**:
- **Per-Node Limits**: Distribute max_active across NUMA nodes
- **Local Execution**: Prefer local node execution when possible
- **Memory Locality**: Optimize memory access patterns
- **Load Distribution**: Balance work across NUMA topology

#### CPU Affinity Scopes
```c
enum wq_affn_scope {
    WQ_AFFN_DFL,                       /* Default affinity */
    WQ_AFFN_CPU,                       /* Per-CPU affinity */
    WQ_AFFN_SMT,                       /* SMT core affinity */
    WQ_AFFN_CACHE,                     /* Cache domain affinity */
    WQ_AFFN_NUMA,                      /* NUMA node affinity */
    WQ_AFFN_SYSTEM,                    /* System-wide affinity */
};
```

### CPU Hotplug Integration

#### CPU Online/Offline Handling
**Process**:
1. **Pool Association**: Associate/disassociate pools with CPUs
2. **Worker Migration**: Migrate workers between pools as needed
3. **Work Redistribution**: Redistribute work to available CPUs
4. **State Synchronization**: Maintain consistent state during transitions

#### Isolation and Housekeeping
```c
/* CPU isolation support */
static cpumask_var_t wq_unbound_cpumask;    /* Allowed CPUs for unbound work */
static cpumask_var_t wq_isolated_cpumask;   /* Isolated CPUs to exclude */
```

**Isolation Features**:
- **CPU Isolation**: Respect CPU isolation for real-time workloads
- **Housekeeping Integration**: Integrate with kernel housekeeping subsystem
- **Dynamic Updates**: Handle isolation changes at runtime
- **Performance Optimization**: Optimize for isolated CPU performance

### Power Management Integration

#### Power Efficient Workqueues
```c
static bool wq_power_efficient = IS_ENABLED(CONFIG_WQ_POWER_EFFICIENT_DEFAULT);
```

**Power Optimizations**:
- **CPU Consolidation**: Consolidate work to fewer CPUs when possible
- **Idle State Preservation**: Avoid waking idle CPUs unnecessarily
- **Frequency Scaling**: Consider CPU frequency scaling policies
- **Energy Efficiency**: Balance performance with energy consumption

### Bottom-Half (BH) Workqueues

#### BH Worker Pools
```c
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS], 
                                     bh_worker_pools);
static DEFINE_PER_CPU_SHARED_ALIGNED(struct irq_work [NR_STD_WORKER_POOLS], 
                                     bh_pool_irq_works);
```

**BH Features**:
- **Interrupt Context**: Execute work in interrupt context
- **Low Latency**: Minimal scheduling latency for critical work
- **CPU Affinity**: Strong CPU affinity for interrupt handling
- **Resource Limits**: Bounded execution time to prevent softirq starvation

### Performance Monitoring and Statistics

#### Work Queue Statistics
```c
enum pool_workqueue_stats {
    PWQ_STAT_STARTED,                  /* Work items started */
    PWQ_STAT_COMPLETED,                /* Work items completed */
    PWQ_STAT_CPU_TIME,                 /* Total CPU time consumed */
    PWQ_STAT_CPU_INTENSIVE,            /* CPU intensive violations */
    PWQ_STAT_CM_WAKEUP,               /* Concurrency mgmt wakeups */
    PWQ_STAT_REPATRIATED,             /* Worker repatriations */
    PWQ_STAT_MAYDAY,                  /* Rescue operations */
    PWQ_STAT_RESCUED,                 /* Rescued work items */
};
```

**Monitoring Capabilities**:
- **Execution Statistics**: Track work execution metrics
- **Performance Analysis**: Analyze workqueue performance patterns
- **Resource Usage**: Monitor resource consumption
- **Debugging Support**: Provide debugging information

#### Watchdog Integration
```c
struct worker_pool {
    unsigned long watchdog_ts;         /* Watchdog timestamp */
    bool         cpu_stall;           /* CPU stall detection */
};
```

**Watchdog Features**:
- **Stall Detection**: Detect hung workers and work items
- **Timeout Monitoring**: Monitor work execution timeouts
- **System Health**: Contribute to overall system health monitoring
- **Recovery Actions**: Trigger recovery actions for stalled work

## Integration Points

### Scheduler Integration
- **Process Flags**: Set PF_WQ_WORKER flag for scheduler awareness
- **Load Balancing**: Coordinate with scheduler load balancing
- **Priority Handling**: Handle work priority through scheduler
- **CPU Affinity**: Integrate with scheduler CPU affinity management

### Memory Management Integration
- **Memory Reclaim**: Support memory reclaim through WQ_MEM_RECLAIM
- **Allocation Context**: Proper GFP flag handling for work contexts
- **NUMA Awareness**: Integrate with NUMA memory allocation policies
- **Resource Limits**: Respect memory limits and constraints

### Interrupt and Timer Integration
- **IRQ Work**: Integration with IRQ work infrastructure
- **Timer Services**: Use kernel timers for worker management
- **Softirq Coordination**: Coordinate with softirq processing
- **Bottom-Half Execution**: Provide bottom-half execution context

### Container and Namespace Integration
- **Cgroup Integration**: Respect cgroup resource limits
- **Namespace Awareness**: Handle namespace constraints
- **Resource Isolation**: Maintain resource isolation between containers
- **Security Boundaries**: Respect security boundaries

This comprehensive workqueue implementation provides the foundation for asynchronous work execution in Linux, enabling efficient, scalable, and manageable background processing while maintaining system responsiveness and resource efficiency across diverse hardware configurations and workload patterns.