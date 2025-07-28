# Linux Kernel Out-of-Memory Killer (`mm/oom_kill.c`)

## Overview

The `mm/oom_kill.c` file implements Linux's Out-of-Memory (OOM) killer, a critical last-resort mechanism that terminates processes when the system runs out of memory. This subsystem prevents complete system lockup by intelligently selecting and killing processes to free memory, ensuring system stability under severe memory pressure. The OOM killer balances system survival with fairness, attempting to kill the least important processes while preserving critical system functionality.

## Core Architecture

### 1. OOM Control Structure

**Central Coordination**:
- **`struct oom_control`**: Contains context for OOM killing decisions
- **Memory Constraints**: Tracks NUMA nodes, memory cgroups, and allocation context
- **Target Selection**: Manages victim selection and killing coordination

**Global Synchronization** - Lines 68-70:
```c
DEFINE_MUTEX(oom_lock);           // Serializes OOM killer invocations
DEFINE_MUTEX(oom_adj_mutex);      // Protects oom_score_adj updates
```

### 2. Configuration Parameters

**System Control Variables** - Lines 56-58:
- `sysctl_panic_on_oom`: Controls system panic behavior on OOM
- `sysctl_oom_kill_allocating_task`: Kill allocating task instead of largest
- `sysctl_oom_dump_tasks`: Enable detailed task information dumps

**OOM Victim Tracking** - Lines 482-485:
```c
static atomic_t oom_victims = ATOMIC_INIT(0);
static DECLARE_WAIT_QUEUE_HEAD(oom_victims_wait);
static bool oom_killer_disabled __read_mostly;
```

## Victim Selection Algorithm

### 1. Eligibility Filtering

**Task Eligibility** - Lines 90-126:
- **`oom_cpuset_eligible()`**: Checks NUMA and cpuset constraints
- **NUMA Policy Validation**: Ensures victim respects memory policies
- **Cpuset Membership**: Validates task belongs to same memory domain

**Unkillable Task Protection** - Lines 163-170:
```c
static bool oom_unkillable_task(struct task_struct *p) {
    if (is_global_init(p))      // Protect init process
        return true;
    if (p->flags & PF_KTHREAD)  // Protect kernel threads
        return true;
    return false;
}
```

### 2. Badness Scoring

**Heuristic Calculation** - Lines 194-200:
- **Memory Usage**: Primary factor in victim selection
- **oom_score_adj**: Administrative adjustment (-1000 to +1000)
- **Predictability**: Simple, deterministic algorithm for consistent behavior
- **Fairness**: Avoids repeatedly killing the same processes

**Scoring Factors**:
- **RSS Memory**: Resident set size as primary indicator
- **Virtual Memory**: Total virtual memory usage consideration
- **Age and Children**: Newer processes preferred over established ones
- **Administrative Override**: Respects manual priority adjustments

## Process Killing Infrastructure

### 1. Kill Process Execution

**`__oom_kill_process()`** - Lines 921-1001:
- **Signal Delivery**: Sends SIGKILL to victim and related processes
- **Memory Reserve Access**: Grants access to emergency memory reserves
- **Process Group Handling**: Kills all threads sharing the same memory space
- **Resource Cleanup**: Ensures proper cleanup of victim resources

**Victim Marking** - Line 953:
```c
mark_oom_victim(victim);  // Grants memory reserves and tracks state
```

### 2. Memory Cgroup Integration

**Group Kill Logic** - Lines 1007-1063:
- **`oom_kill_memcg_member()`**: Kills all eligible tasks in memory cgroup
- **OOM Groups**: Implements cgroup-wide killing policies
- **Hierarchical Killing**: Can kill entire cgroup hierarchies
- **Event Notification**: Reports cgroup OOM events

## OOM Reaper

### 1. Asynchronous Memory Reclaim

**OOM Reaper Thread** - Lines 510-513:
```c
static struct task_struct *oom_reaper_th;
static DECLARE_WAIT_QUEUE_HEAD(oom_reaper_wait);
static struct task_struct *oom_reaper_list;
```

**Purpose**: Asynchronously reclaims memory from killed processes to prevent OOM killer from getting stuck waiting for memory to be freed.

### 2. Memory Reclamation Process

**`__oom_reap_task_mm()`** - Lines 515-563:
- **VMA Iteration**: Scans virtual memory areas of victim process
- **Anonymous Page Focus**: Prioritizes easily reclaimable anonymous pages
- **MMU Notifier Integration**: Coordinates with external memory users
- **Partial Reclaim**: Handles cases where complete reclaim isn't possible

**MMF Flag Management**:
- **`MMF_UNSTABLE`**: Marks memory as unstable during reaping
- **`MMF_OOM_SKIP`**: Prevents further reaping attempts
- **Race Prevention**: Coordinates with process exit paths

## Diagnostic and Debugging

### 1. Memory State Reporting

**`dump_tasks()`** - Lines 425-445:
- **Process Information**: PID, UID, memory usage statistics
- **Memory Breakdown**: RSS, anonymous, file, shared memory details
- **OOM Score Display**: Shows current badness scores
- **Selective Dumping**: Respects memory cgroup and NUMA constraints

**System State Dumps** - Lines 459-477:
```c
dump_header() {
    // GFP mask and allocation context
    // Stack trace of allocation path
    // Memory statistics and free memory
    // Task list with memory usage
}
```

### 2. Unreclaimable Slab Detection

**`should_dump_unreclaim_slab()`** - Lines 178-191:
- **Slab vs. User Memory**: Compares unreclaimable slab to user memory
- **Kernel Memory Issues**: Identifies kernel memory leaks as OOM cause
- **Diagnostic Trigger**: Enables detailed slab dump when appropriate

## Notification and Extension Points

### 1. OOM Notifier Chain

**External Integration** - Lines 1089-1099:
```c
static BLOCKING_NOTIFIER_HEAD(oom_notify_list);
int register_oom_notifier(struct notifier_block *nb);
int unregister_oom_notifier(struct notifier_block *nb);
```

**Use Cases**:
- **Custom Handlers**: Allow subsystems to respond to OOM conditions
- **Logging Integration**: External logging and monitoring systems
- **Recovery Actions**: Implement custom recovery mechanisms

### 2. Panic Integration

**`check_panic_on_oom()`** - Lines 1068-1087:
- **System-wide Panic**: Option to panic instead of killing processes
- **Constraint-based Control**: Different behavior for different OOM types
- **SysRq Exception**: Prevents panic for manual OOM triggers

## Memory Constraint Handling

### 1. NUMA Awareness

**Node-specific OOM**:
- **Local Node Pressure**: Handles memory pressure on specific NUMA nodes
- **Memory Policy Respect**: Considers process memory binding policies
- **Cross-node Allocation**: Manages memory allocation across NUMA topology

### 2. Memory Cgroup Integration

**Hierarchical OOM**:
- **Per-cgroup Limits**: Respects memory cgroup resource limits
- **Group Policies**: Implements group-wide OOM killing policies
- **Resource Accounting**: Accurate memory accounting per cgroup

## Performance and Scalability

### 1. Lock-free Operations

**Atomic Victim Tracking**:
- **`oom_victims` Counter**: Tracks active OOM victims without locks
- **Wait Queue Management**: Efficient waiting for victim cleanup
- **Memory Barriers**: Ensures proper ordering of victim state updates

### 2. Efficient Task Scanning

**Process Iteration Optimization**:
- **RCU Protection**: Safe concurrent access to process lists
- **Softlockup Prevention**: Periodic yielding during long scans
- **Early Termination**: Stops scanning when suitable victim found

## Security Considerations

### 1. Privilege Escalation Prevention

**Process Protection**:
- **Init Process Protection**: Never kills PID 1 (init)
- **Kernel Thread Protection**: Protects essential kernel threads
- **Administrative Control**: Respects oom_score_adj settings

### 2. Resource Exhaustion Protection

**Emergency Reserves**:
- **Memory Reserve Access**: Provides emergency memory to dying processes
- **Priority Access**: Ensures OOM victims can complete exit
- **Reserve Depletion Prevention**: Prevents victims from consuming all reserves

## Error Handling and Recovery

### 1. Graceful Degradation

**Fallback Mechanisms**:
- **Reaper Failure Handling**: Continues operation if reaper fails
- **Partial Success**: Handles partial memory reclamation gracefully
- **Retry Logic**: Implements intelligent retry mechanisms

### 2. State Consistency

**Process State Management**:
- **Exit Race Handling**: Coordinates with process exit paths
- **Memory Management Integration**: Ensures consistency with MM subsystem
- **Signal Delivery**: Reliable signal delivery to victims

## Integration with MM Subsystem

### 1. Page Allocator Integration

**Allocation Path Integration**:
- **Last Resort Activation**: Triggered only when allocation fails
- **GFP Flag Respect**: Honors allocation flags and constraints
- **Order-aware Decisions**: Considers allocation order in decisions

### 2. Memory Reclaim Coordination

**Reclaim Integration**:
- **Shrinker Coordination**: Works with memory shrinkers
- **LRU Management**: Coordinates with LRU page reclaim
- **Compaction Integration**: Works with memory compaction

## Debugging and Observability

### 1. Tracing Support

**Event Tracing**:
- **OOM Events**: Detailed tracing of OOM killer activation
- **Victim Selection**: Traces victim selection process
- **Reaper Activity**: Monitors reaper thread activity

### 2. Statistics and Monitoring

**Kernel Statistics**:
- **`OOM_KILL` Events**: VM event counters for OOM kills
- **Memory Cgroup Events**: Per-cgroup OOM statistics
- **Victim Tracking**: Active victim count monitoring

The OOM killer represents a critical balance between system stability and process fairness. Its sophisticated victim selection algorithm, combined with robust error handling and integration with the broader memory management subsystem, ensures that Linux systems can survive severe memory pressure while minimizing the impact on important processes and system functionality.