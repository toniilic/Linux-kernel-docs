# kernel/kthread.c - Linux Kernel Thread Management

## Overview

This file implements the kernel thread (kthread) subsystem for Linux, originally developed by Rusty Russell at IBM Corporation in 2004 and enhanced by Red Hat Inc. in 2009. It provides a comprehensive framework for creating, managing, and controlling kernel-level threads that run entirely in kernel space. The kthread subsystem enables the kernel to perform background tasks, handle asynchronous operations, and implement various kernel services through dedicated execution contexts.

## Historical Development

### Key Contributors and Evolution
- **Rusty Russell - IBM Corporation (2004)**: Original kthread framework design
- **Red Hat Inc. (2009)**: Major enhancements and improvements
- **Community Contributors**: Ongoing development and feature additions

### Design Philosophy
The kthread system is designed around the principle of using a dedicated kernel thread (`kthreadd`) to create other kernel threads, ensuring a clean environment even when invoked from user space contexts (such as module loading, CPU hotplug operations, etc.).

## Core Concepts

### Kernel Thread Architecture

#### Thread Creation Flow
```
User/Kernel Request → kthread_create() → kthreadd Process → New Kernel Thread
        ↓                    ↓                ↓                    ↓
   [Requestor]         [Creation API]    [Thread Factory]    [Worker Thread]
```

#### Thread Lifecycle States
- **Created**: Thread created but not started
- **Running**: Thread actively executing
- **Parked**: Thread temporarily suspended
- **Stopping**: Thread being requested to stop
- **Exited**: Thread completed execution

### Thread Types and Categories

#### Generic Kernel Threads
- **Worker Threads**: General purpose background processing
- **Service Threads**: Dedicated kernel service handlers
- **Maintenance Threads**: System maintenance and cleanup tasks

#### Per-CPU Threads
- **CPU-Bound Tasks**: Threads bound to specific CPUs
- **NUMA-Aware**: Threads with NUMA locality preferences
- **CPU Hotplug**: Threads that handle CPU online/offline events

#### Specialized Threads
- **Freezable Threads**: Can be frozen during system suspend
- **Non-Freezable**: Critical threads that cannot be suspended
- **RT Priority**: Real-time priority kernel threads

## Key Data Structures

### `struct kthread` - Kernel Thread Descriptor
```c
struct kthread {
    unsigned long flags;                /* Thread state flags */
    unsigned int cpu;                   /* CPU binding */
    unsigned int node;                  /* NUMA node preference */
    int started;                        /* Thread started flag */
    int result;                         /* Thread exit result */
    int (*threadfn)(void *);           /* Thread function pointer */
    void *data;                         /* Thread function data */
    struct completion parked;           /* Parking synchronization */
    struct completion exited;           /* Exit synchronization */
#ifdef CONFIG_BLK_CGROUP
    struct cgroup_subsys_state *blkcg_css; /* Block cgroup context */
#endif
    char *full_name;                    /* Full thread name */
    struct task_struct *task;           /* Associated task structure */
    struct list_head hotplug_node;     /* CPU hotplug list */
    struct cpumask *preferred_affinity; /* Preferred CPU affinity */
};
```

### `struct kthread_create_info` - Thread Creation Request
```c
struct kthread_create_info {
    /* Information passed to kthread() from kthreadd */
    char *full_name;                    /* Thread name */
    int (*threadfn)(void *data);        /* Thread function */
    void *data;                         /* Function data */
    int node;                           /* NUMA node */
    
    /* Result passed back to kthread_create() from kthreadd */
    struct task_struct *result;         /* Created task */
    struct completion *done;            /* Creation completion */
    
    struct list_head list;              /* Creation queue linkage */
};
```

### Thread State Flags
```c
enum KTHREAD_BITS {
    KTHREAD_IS_PER_CPU = 0,            /* Per-CPU thread flag */
    KTHREAD_SHOULD_STOP,               /* Stop request flag */
    KTHREAD_SHOULD_PARK,               /* Park request flag */
};
```

### Global Thread Management
```c
static DEFINE_SPINLOCK(kthread_create_lock);    /* Creation serialization */
static LIST_HEAD(kthread_create_list);          /* Pending creation requests */
struct task_struct *kthreadd_task;              /* Thread factory task */

static LIST_HEAD(kthreads_hotplug);             /* CPU hotplug thread list */
static DEFINE_MUTEX(kthreads_hotplug_lock);     /* Hotplug synchronization */
```

## Core Functions

### Thread Creation and Management

#### `kthread_create_on_node()` - Create Kernel Thread
```c
struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
                                           void *data, int node,
                                           const char namefmt[], ...)
```

**Purpose**: Create a new kernel thread with NUMA node preference

**Creation Process**:
1. **Request Preparation**: Allocate and initialize creation request structure
2. **Name Formatting**: Format thread name using printf-style arguments
3. **Queue Request**: Add creation request to kthreadd's work queue
4. **Wake kthreadd**: Signal the thread factory to process request
5. **Wait for Completion**: Wait for thread creation to complete
6. **Return Result**: Return created task structure or error

**Key Features**:
- **NUMA Awareness**: Allocate thread structures on specified node
- **Killable Wait**: Can be interrupted by fatal signals during creation
- **Error Handling**: Comprehensive error handling and cleanup
- **Name Management**: Full printf-style name formatting support

#### `__kthread_create_on_node()` - Internal Creation Implementation
```c
static struct task_struct *__kthread_create_on_node(int (*threadfn)(void *data),
                                                    void *data, int node,
                                                    const char namefmt[],
                                                    va_list args)
```

**Internal Creation Steps**:
```c
/* Allocate creation request */
create = kmalloc(sizeof(*create), GFP_KERNEL);
create->threadfn = threadfn;
create->data = data;
create->node = node;
create->done = &done;
create->full_name = kvasprintf(GFP_KERNEL, namefmt, args);

/* Queue request for kthreadd */
spin_lock(&kthread_create_lock);
list_add_tail(&create->list, &kthread_create_list);
spin_unlock(&kthread_create_lock);

/* Wake up kthreadd and wait for completion */
wake_up_process(kthreadd_task);
wait_for_completion_killable(&done);
```

### Thread Binding and Affinity

#### `kthread_bind()` - Bind Thread to CPU
```c
void kthread_bind(struct task_struct *p, unsigned int cpu)
```

**CPU Binding Process**:
1. **Inactive Verification**: Ensure thread is not currently running
2. **Affinity Setting**: Set CPU affinity to specified CPU
3. **No-Setaffinity Flag**: Prevent user space from changing affinity
4. **State Validation**: Verify thread hasn't started yet

#### `kthread_bind_mask()` - Bind Thread to CPU Mask
```c
void kthread_bind_mask(struct task_struct *p, const struct cpumask *mask)
```

**Mask Binding Features**:
- **Multi-CPU Support**: Bind to multiple CPUs simultaneously
- **NUMA Optimization**: Consider NUMA topology in binding
- **Hotplug Integration**: Handle CPU hotplug events appropriately
- **Load Balancing**: Allow scheduler to balance within mask

#### `kthread_create_on_cpu()` - Create CPU-Bound Thread
```c
struct task_struct *kthread_create_on_cpu(int (*threadfn)(void *data),
                                          void *data, unsigned int cpu,
                                          const char *namefmt)
```

**CPU-Specific Creation**:
```c
/* Create thread on appropriate NUMA node */
p = kthread_create_on_node(threadfn, data, cpu_to_node(cpu), 
                          namefmt, cpu);

/* Bind to specific CPU */
kthread_bind(p, cpu);

/* Mark as per-CPU thread */
to_kthread(p)->cpu = cpu;
```

### Thread Control and Lifecycle

#### `kthread_should_stop()` - Check Stop Request
```c
bool kthread_should_stop(void)
```

**Stop Detection**:
- **Flag Check**: Check KTHREAD_SHOULD_STOP flag
- **Non-Blocking**: Immediate return without sleeping
- **Thread Safety**: Safe to call from any kernel context
- **Usage Pattern**: Typically used in thread main loop

#### `kthread_stop()` - Request Thread Termination
```c
int kthread_stop(struct task_struct *k)
```

**Stop Process**:
1. **Stop Request**: Set KTHREAD_SHOULD_STOP flag
2. **Thread Wake-up**: Wake up thread if sleeping
3. **Exit Wait**: Wait for thread to acknowledge stop
4. **Result Collection**: Collect thread's exit result
5. **Cleanup**: Clean up thread resources

#### `kthread_park()` and `kthread_unpark()` - Thread Parking
```c
int kthread_park(struct task_struct *k)
void kthread_unpark(struct task_struct *k)
```

**Parking Mechanism**:
- **Temporary Suspension**: Suspend thread without termination
- **State Preservation**: Maintain thread state during parking
- **Quick Resume**: Fast resume when unparked
- **Synchronization**: Proper synchronization with thread execution

#### `__kthread_parkme()` - Internal Parking Implementation
```c
static void __kthread_parkme(struct kthread *self)
```

**Parking Process**:
```c
for (;;) {
    /* Set special parked state */
    set_special_state(TASK_PARKED);
    
    /* Check if still should be parked */
    if (!test_bit(KTHREAD_SHOULD_PARK, &self->flags))
        break;
    
    /* Notify parker and sleep */
    preempt_disable();
    complete(&self->parked);
    schedule_preempt_disabled();
    preempt_enable();
}
__set_current_state(TASK_RUNNING);
```

### Thread Exit and Cleanup

#### `kthread_exit()` - Thread Termination
```c
void __noreturn kthread_exit(long result)
```

**Exit Process**:
1. **Result Storage**: Store exit result for kthread_stop()
2. **Hotplug Cleanup**: Remove from CPU hotplug lists
3. **Affinity Cleanup**: Free preferred affinity mask
4. **Process Exit**: Call do_exit() to terminate

#### `kthread_complete_and_exit()` - Completion and Exit
```c
void __noreturn kthread_complete_and_exit(struct completion *comp, long code)
```

**Combined Operation**:
- **Completion Signal**: Signal completion before exit
- **Module Safety**: Safe for use in loadable modules
- **Synchronization**: Proper synchronization with waiters
- **Clean Exit**: Guaranteed clean thread termination

### NUMA and CPU Affinity Management

#### `kthread_affine_node()` - NUMA Node Affinity
```c
static void kthread_affine_node(void)
```

**NUMA Affinity Process**:
1. **Node Validation**: Verify NUMA node availability
2. **Housekeeping Integration**: Respect CPU isolation settings
3. **Hotplug Registration**: Register for CPU hotplug events
4. **Affinity Update**: Set appropriate CPU affinity

#### `kthread_fetch_affinity()` - Affinity Calculation
```c
static void kthread_fetch_affinity(struct kthread *kthread, struct cpumask *cpumask)
```

**Affinity Logic**:
```c
if (kthread->preferred_affinity) {
    pref = kthread->preferred_affinity;
} else {
    pref = cpumask_of_node(kthread->node);
}

/* Intersect with housekeeping CPUs */
cpumask_and(cpumask, pref, housekeeping_cpumask(HK_TYPE_KTHREAD));

/* Fallback to all housekeeping CPUs if empty */
if (cpumask_empty(cpumask))
    cpumask_copy(cpumask, housekeeping_cpumask(HK_TYPE_KTHREAD));
```

### Freezer Integration

#### `kthread_freezable_should_stop()` - Freezable Thread Support
```c
bool kthread_freezable_should_stop(bool *was_frozen)
```

**Freezer Integration**:
1. **Freeze Check**: Check if system is freezing
2. **Refrigerator Entry**: Enter refrigerator if needed
3. **Stop Check**: Check stop request after thawing
4. **State Reporting**: Report freeze state to caller

**Freezer Safety**:
```c
if (unlikely(freezing(current)))
    frozen = __refrigerator(true);
    
if (was_frozen)
    *was_frozen = frozen;
    
return kthread_should_stop();
```

## Advanced Features

### Per-CPU Thread Management

#### Per-CPU Thread Properties
```c
void kthread_set_per_cpu(struct task_struct *k, int cpu)
bool kthread_is_per_cpu(struct task_struct *p)
```

**Per-CPU Features**:
- **CPU Binding**: Strong binding to specific CPU
- **Hotplug Handling**: Automatic CPU hotplug management
- **NUMA Optimization**: Optimal NUMA locality
- **Isolation Support**: Respect CPU isolation settings

### CPU Hotplug Integration

#### Hotplug Thread Management
```c
static LIST_HEAD(kthreads_hotplug);
static DEFINE_MUTEX(kthreads_hotplug_lock);
```

**Hotplug Coordination**:
- **Registration**: Register threads for hotplug events
- **Affinity Updates**: Update affinity during CPU changes
- **Migration Support**: Migrate threads during CPU offline
- **State Synchronization**: Synchronize state across hotplug events

### Block Cgroup Integration

#### Cgroup Context Preservation
```c
#ifdef CONFIG_BLK_CGROUP
struct cgroup_subsys_state *blkcg_css;
#endif
```

**Cgroup Features**:
- **Context Inheritance**: Inherit block cgroup context
- **I/O Accounting**: Proper I/O accounting in cgroups
- **Resource Limits**: Respect cgroup resource limits
- **Policy Enforcement**: Enforce cgroup policies

### Thread Introspection and Debugging

#### Thread Information Access
```c
void *kthread_func(struct task_struct *task)
void *kthread_data(struct task_struct *task)
void *kthread_probe_data(struct task_struct *task)
```

**Introspection Features**:
- **Function Identification**: Identify thread function
- **Data Access**: Access thread-specific data
- **Safe Probing**: Safe data access with fault handling
- **Debugging Support**: Support for kernel debugging tools

#### Thread Name Management
```c
char *full_name;  /* Full thread name storage */
```

**Name Features**:
- **Full Name Storage**: Store complete thread names
- **Truncation Handling**: Handle comm field truncation
- **Format Support**: Printf-style name formatting
- **Debugging Aid**: Assist in debugging and monitoring

## Thread Patterns and Best Practices

### Common Thread Patterns

#### Worker Thread Pattern
```c
static int worker_thread(void *data)
{
    while (!kthread_should_stop()) {
        /* Wait for work */
        wait_event_interruptible(work_queue, 
                                has_work() || kthread_should_stop());
        
        if (kthread_should_stop())
            break;
            
        /* Process work */
        process_work();
    }
    
    return 0;
}
```

#### Service Thread Pattern
```c
static int service_thread(void *data)
{
    while (!kthread_should_stop()) {
        /* Service-specific processing */
        service_process();
        
        /* Check for freezing */
        try_to_freeze();
        
        /* Sleep or wait for events */
        if (kthread_should_stop())
            break;
            
        schedule_timeout_interruptible(timeout);
    }
    
    return 0;
}
```

#### Freezable Thread Pattern
```c
static int freezable_thread(void *data)
{
    bool was_frozen;
    
    while (!kthread_freezable_should_stop(&was_frozen)) {
        /* Handle freeze state if needed */
        if (was_frozen) {
            /* Post-freeze recovery */
            recover_from_freeze();
        }
        
        /* Normal processing */
        do_work();
    }
    
    return 0;
}
```

### Thread Creation Helpers

#### Simple Thread Creation
```c
#define kthread_run(threadfn, data, namefmt, ...)                      \
({                                                                      \
    struct task_struct *__k                                            \
        = kthread_create(threadfn, data, namefmt, ## __VA_ARGS__);     \
    if (!IS_ERR(__k))                                                  \
        wake_up_process(__k);                                          \
    __k;                                                               \
})
```

#### NUMA-Aware Creation
```c
#define kthread_create(threadfn, data, namefmt, ...)                   \
    kthread_create_on_node(threadfn, data, NUMA_NO_NODE, namefmt, ## __VA_ARGS__)
```

## Performance Optimizations

### Creation Optimization
- **Dedicated Factory**: Use kthreadd for clean creation environment
- **NUMA Locality**: Create threads on appropriate NUMA nodes
- **Resource Pooling**: Reuse thread structures where possible
- **Fast Path**: Optimize common creation patterns

### Scheduling Optimization
- **CPU Affinity**: Optimal CPU binding for performance
- **Priority Management**: Appropriate priority assignment
- **Load Balancing**: Balance threads across CPUs
- **Cache Locality**: Maintain cache locality where possible

### Memory Management
- **Stack Allocation**: Optimal stack allocation and management
- **Structure Reuse**: Reuse kthread structures when possible
- **NUMA Awareness**: Allocate on appropriate NUMA nodes
- **Memory Reclaim**: Proper memory cleanup on exit

## Error Handling and Edge Cases

### Creation Failures
- **Memory Exhaustion**: Handle out-of-memory conditions
- **Invalid Parameters**: Validate thread creation parameters
- **Signal Interruption**: Handle signal interruption during creation
- **Resource Limits**: Respect system resource limits

### Runtime Error Handling
- **Unexpected Exit**: Handle unexpected thread termination
- **Signal Handling**: Proper signal handling in threads
- **Resource Cleanup**: Ensure complete resource cleanup
- **State Consistency**: Maintain consistent thread state

### CPU Hotplug Edge Cases
- **Offline CPUs**: Handle threads on offline CPUs
- **Migration Failures**: Handle failed thread migration
- **Affinity Conflicts**: Resolve CPU affinity conflicts
- **Timing Races**: Handle hotplug timing races

## Integration Points

### Scheduler Integration
- **Task Creation**: Integration with process creation
- **CPU Binding**: Scheduler CPU affinity management
- **Priority Handling**: Thread priority management
- **Load Balancing**: Integration with load balancing

### Memory Management Integration
- **Stack Management**: Kernel stack allocation and management
- **NUMA Placement**: NUMA-aware memory placement
- **Resource Accounting**: Memory resource accounting
- **Cleanup Coordination**: Coordinate with memory management cleanup

### Power Management Integration
- **CPU Idle**: Coordinate with CPU idle management
- **Suspend/Resume**: Support system suspend and resume
- **Freezer Framework**: Integration with kernel freezer
- **Power Domains**: Consider power domain constraints

### Container and Namespace Integration
- **Cgroup Support**: Integration with control groups
- **Namespace Handling**: Proper namespace handling
- **Resource Isolation**: Maintain resource isolation
- **Security Boundaries**: Respect security boundaries

This comprehensive kthread implementation provides the foundation for kernel-level threading in Linux, enabling efficient, scalable, and manageable background processing while maintaining system stability and performance across diverse hardware configurations and workload patterns.