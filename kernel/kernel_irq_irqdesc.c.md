# kernel/irq/irqdesc.c - Linux Kernel Interrupt Descriptor Management

## Overview

This file implements the core interrupt descriptor infrastructure for Linux, providing a sophisticated abstraction layer for managing interrupts across diverse hardware platforms. The interrupt descriptor system serves as the central registry for all interrupts in the system, managing everything from simple GPIO interrupts to complex MSI/MSI-X message-signaled interrupts. Originally designed for simple interrupt controllers, it has evolved to support modern features like hierarchical interrupt domains, dynamic allocation, NUMA awareness, and comprehensive statistics collection.

## Historical Development

### Key Evolution Points
- **Early Linux (1991-1995)**: Simple static interrupt descriptor arrays
- **SMP Era (1996-2005)**: Per-CPU data structures and lock-based synchronization
- **Scalability Era (2006-2015)**: Sparse IRQ allocation and RCU-based lockless operations
- **Modern Era (2016-present)**: IRQ domain hierarchies, managed IRQs, and advanced power management

### Design Philosophy
The IRQ descriptor system embodies hardware abstraction through software layers, providing a unified interface for interrupt management regardless of the underlying hardware. It emphasizes scalability from embedded systems with few interrupts to large servers with thousands of interrupt sources, while maintaining performance through sophisticated caching and lock-free algorithms.

## Core Architecture

### Interrupt Management Pipeline
```
Hardware IRQ → IRQ Domain → IRQ Descriptor → IRQ Handler → Device Driver
      ↓            ↓            ↓              ↓           ↓
[HW Controller] [Mapping]   [Management]   [Flow Control] [Action]
```

### Descriptor Organization Models
```
Dense Mode: Fixed Array (Legacy)
IRQ Number → Direct Array Index → IRQ Descriptor

Sparse Mode: Maple Tree (Modern) 
IRQ Number → Maple Tree Lookup → IRQ Descriptor
```

## Key Data Structures

### Core Interrupt Descriptor
```c
struct irq_desc {
    struct irq_common_data  irq_common_data;  /* Common IRQ data */
    struct irq_data         irq_data;         /* IRQ chip data */
    struct irqstat __percpu *kstat_irqs;      /* Per-CPU statistics */
    irq_flow_handler_t      handle_irq;       /* Flow handler function */
    struct irqaction        *action;          /* Action chain */
    unsigned int            status_use_accessors; /* Status flags */
    unsigned int            depth;            /* Nested disable depth */
    unsigned int            wake_depth;       /* Nested wake disable depth */
    unsigned int            irq_count;        /* Spurious interrupt count */
    unsigned long           last_unhandled;   /* Last spurious timestamp */
    unsigned int            irqs_unhandled;   /* Unhandled count */
    atomic_t                threads_active;   /* Active threaded handlers */
    wait_queue_head_t       wait_for_threads; /* Thread completion wait */
    raw_spinlock_t          lock;             /* Descriptor protection */
    struct mutex            request_mutex;    /* Setup/teardown serialization */
    int                     parent_irq;       /* Parent IRQ for domains */
    struct module          *owner;            /* Module ownership */
    const char             *name;             /* IRQ name */
    struct kobject          kobj;             /* sysfs representation */
    struct dentry          *debugfs_file;     /* debugfs entry */
    struct rcu_head         rcu;              /* RCU cleanup */
    /* SMP-specific fields */
    struct cpumask         *pending_mask;     /* Pending affinity */
    /* NUMA-specific fields */
    int                     numa_node;        /* NUMA node assignment */
    /* ... */
} ____cacheline_internodealigned_in_smp;
```

### IRQ Statistics Structure
```c
struct irqstat {
    unsigned int            cnt;              /* Real-time interrupt count */
#ifdef CONFIG_GENERIC_IRQ_STAT_SNAPSHOT
    unsigned int            ref;              /* Snapshot reference */
#endif
};
```

### Sparse IRQ Infrastructure
```c
static DEFINE_MUTEX(sparse_irq_lock);
static struct maple_tree sparse_irqs = MTREE_INIT_EXT(sparse_irqs,
                    MT_FLAGS_ALLOC_RANGE |
                    MT_FLAGS_LOCK_EXTERN |
                    MT_FLAGS_USE_RCU,
                    sparse_irq_lock);
```

### IRQ State Definitions
```c
/* IRQ data state flags (irq_data.state) */
enum {
    IRQD_TRIGGER_MASK               = 0x0000000f,
    IRQD_SETAFFINITY_PENDING        = (1 << 8),
    IRQD_NO_BALANCING               = (1 << 10),
    IRQD_PER_CPU                    = (1 << 11),
    IRQD_AFFINITY_SET               = (1 << 12),
    IRQD_LEVEL                      = (1 << 13),
    IRQD_WAKEUP_STATE               = (1 << 14),
    IRQD_MOVE_PCNTXT                = (1 << 15),
    IRQD_IRQ_DISABLED               = (1 << 16),
    IRQD_IRQ_MASKED                 = (1 << 17),
    IRQD_IRQ_INPROGRESS             = (1 << 18),
    IRQD_WAKEUP_ARMED               = (1 << 19),
    IRQD_FORWARDED_TO_VCPU          = (1 << 20),
    IRQD_AFFINITY_MANAGED           = (1 << 21),
    IRQD_IRQ_STARTED                = (1 << 22),
    IRQD_MANAGED_SHUTDOWN           = (1 << 23),
    IRQD_SINGLE_TARGET              = (1 << 24),
    IRQD_DEFAULT_TRIGGER_SET        = (1 << 25),
    IRQD_CAN_RESERVE                = (1 << 26),
    IRQD_HANDLE_ENFORCE_IRQCTX      = (1 << 27),
    IRQD_AFFINITY_ON_ACTIVATE       = (1 << 28),
    IRQD_IRQ_ENABLED_ON_SUSPEND     = (1 << 29),
    IRQD_RESEND_WHEN_IN_PROGRESS    = (1 << 30),
};

/* IRQ descriptor internal state flags (desc.istate) */
enum {
    IRQS_AUTODETECT     = 0x00000001,  /* Auto-detection active */
    IRQS_SPURIOUS_DISABLED = 0x00000002, /* Disabled due to spurious */
    IRQS_POLL_INPROGRESS = 0x00000008, /* Polling in progress */
    IRQS_ONESHOT        = 0x00000020,  /* One-shot handling */
    IRQS_REPLAY         = 0x00000040,  /* Replay pending */
    IRQS_WAITING        = 0x00000080,  /* Waiting for response */
    IRQS_PENDING        = 0x00000200,  /* IRQ pending resend */
    IRQS_SUSPENDED      = 0x00000800,  /* IRQ suspended */
    IRQS_TIMINGS        = 0x00001000,  /* Timing analysis enabled */
    IRQS_NMI            = 0x00002000,  /* NMI delivery */
};
```

## Core Functions

### IRQ Descriptor Allocation and Management

#### `early_irq_init()` - Boot-time Initialization
```c
int __init early_irq_init(void)
```

**Purpose**: Initialize IRQ descriptor infrastructure during early boot

**Process Flow**:
1. **IRQ Count Determination**: Call `arch_probe_nr_irqs()` to determine platform IRQ requirements
2. **Memory Allocation**: Allocate initial descriptor storage (dense or sparse mode)
3. **Default Initialization**: Initialize descriptors with safe defaults
4. **Architecture Integration**: Set up platform-specific IRQ infrastructure

**Implementation Details**:
```c
int __init early_irq_init(void)
{
    int count;
    int i;

    count = arch_probe_nr_irqs();
    printk(KERN_INFO "NR_IRQS: %d, nr_irqs: %d, preallocated irqs: %d\n",
           NR_IRQS, count, initcnt);
    
#ifdef CONFIG_SPARSE_IRQ
    /* Initialize maple tree for sparse IRQ allocation */
    mtree_init(&sparse_irqs);
#else
    /* Initialize static descriptor array */
    for (i = 0; i < count; i++) {
        desc_set_defaults(i, &irq_desc[i], numa_node_id(), NULL, NULL);
    }
#endif
    
    return arch_early_irq_init();
}
```

#### `irq_to_desc()` - IRQ Number to Descriptor Mapping
```c
struct irq_desc *irq_to_desc(unsigned int irq)
```

**Dual Implementation Strategy**:

**Sparse Mode (CONFIG_SPARSE_IRQ)**:
```c
struct irq_desc *irq_to_desc(unsigned int irq)
{
    return mtree_load(&sparse_irqs, irq);
}
```

**Dense Mode (Legacy)**:
```c
struct irq_desc *irq_to_desc(unsigned int irq)
{
    return (irq < NR_IRQS) ? irq_desc + irq : NULL;
}
```

**Performance Characteristics**:
- **Sparse mode**: O(log n) lookup with excellent cache behavior
- **Dense mode**: O(1) array access but limited to NR_IRQS
- **RCU protection**: Lockless reads during interrupt handling

#### `__irq_alloc_descs()` - Dynamic IRQ Allocation
```c
int __irq_alloc_descs(int irq, unsigned int from, unsigned int cnt, int node,
                      struct module *owner, 
                      const struct irq_affinity_desc *affinity)
```

**Allocation Process**:
1. **Range Location**: Find free IRQ number range using `irq_find_free_area()`
2. **Descriptor Creation**: Allocate and initialize descriptor structures
3. **NUMA Optimization**: Allocate on specified NUMA node
4. **Affinity Setup**: Configure CPU affinity for managed interrupts
5. **Registration**: Add to sparse tree and sysfs hierarchy

**Key Features**:
```c
static int alloc_descs(unsigned int start, unsigned int cnt, int node,
                       const struct irq_affinity_desc *affinity,
                       struct module *owner)
{
    for (i = 0; i < cnt; i++) {
        desc = alloc_desc(start + i, node, flags, mask, owner);
        if (!desc)
            goto err;
        
        mtree_store(&sparse_irqs, start + i, desc, GFP_KERNEL);
        irq_sysfs_add(start + i, desc);
    }
    return start;
}
```

### Locking and Concurrency Control

#### IRQ Descriptor Locking Infrastructure
```c
/* Lock acquisition with guards */
#define scoped_guard(name, args...)                                     \
    for (CLASS(name, scope)(args);                                     \
         __guard_ptr(name)(&scope) || !__is_cond_ptr(name);           \
         __guard_ptr(name)(&scope) = NULL)

/* IRQ descriptor guard definitions */
DEFINE_GUARD(irqdesc_lock, struct irq_desc *, 
             __irq_put_desc_unlock(_T->lock, _T->flags, _T->bus),
             unsigned long flags; bool bus)
```

#### Hierarchical Locking Strategy
1. **Sparse IRQ Mutex** (`sparse_irq_lock`): Protects descriptor allocation/deallocation
2. **IRQ Descriptor Spinlock** (`desc->lock`): Per-descriptor atomic operations
3. **Request Mutex** (`desc->request_mutex`): Setup/teardown serialization
4. **Bus Locks**: Hardware-specific controller access serialization

#### RCU-Protected Operations
```c
static void free_desc(unsigned int irq)
{
    struct irq_desc *desc = irq_to_desc(irq);
    
    unregister_irq_proc(irq, desc);
    irq_sysfs_del(desc);
    delete_irq_desc(irq);
    
    call_rcu(&desc->rcu, delayed_free_desc);
}

static void delayed_free_desc(struct rcu_head *rhp)
{
    struct irq_desc *desc = container_of(rhp, struct irq_desc, rcu);
    kobject_put(&desc->kobj);
}
```

### State Management and Flow Control

#### IRQ Enable/Disable with Reference Counting
```c
void __disable_irq(struct irq_desc *desc)
{
    if (!desc->depth++)
        irq_disable(desc);
}

void __enable_irq(struct irq_desc *desc)
{
    switch (desc->depth) {
    case 0:
        WARN(1, "Unbalanced enable for IRQ %d\n", irq_desc_get_irq(desc));
        break;
    case 1:
        irq_startup(desc, IRQ_RESEND, IRQ_START_FORCE);
        break;
    default:
        desc->depth--;
    }
}
```

#### Flow Handler Types
```c
/* Primary flow handlers */
void handle_level_irq(struct irq_desc *desc);     /* Level-triggered */
void handle_edge_irq(struct irq_desc *desc);      /* Edge-triggered */
void handle_simple_irq(struct irq_desc *desc);    /* Software-decoded */
void handle_bad_irq(struct irq_desc *desc);       /* Uninitialized */
```

**Flow Control Process**:
```c
irqreturn_t handle_irq_event(struct irq_desc *desc)
{
    desc->istate &= ~IRQS_PENDING;
    irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);
    raw_spin_unlock(&desc->lock);
    
    ret = handle_irq_event_percpu(desc);
    
    raw_spin_lock(&desc->lock);
    irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);
    return ret;
}
```

### Statistics Collection and Monitoring

#### Per-CPU Statistics Infrastructure
```c
static inline void kstat_incr_irqs_this_cpu(struct irq_desc *desc)
{
    __this_cpu_inc(desc->kstat_irqs->cnt);
    __this_cpu_inc(kstat.irqs_sum);
    desc->tot_count++;
}

unsigned int kstat_irqs_cpu(unsigned int irq, int cpu)
{
    struct irq_desc *desc = irq_to_desc(irq);
    return desc && desc->kstat_irqs ? 
           per_cpu(desc->kstat_irqs->cnt, cpu) : 0;
}
```

#### Statistics Aggregation
```c
unsigned int kstat_irqs_usr(unsigned int irq)
{
    unsigned int sum;
    
    rcu_read_lock();
    sum = kstat_irqs(irq);
    rcu_read_unlock();
    return sum;
}
```

#### Spurious Interrupt Detection
```c
static void note_interrupt(struct irq_desc *desc, irqreturn_t action_ret)
{
    if (desc->istate & IRQS_POLL_INPROGRESS ||
        irq_settings_is_polled(desc))
        return;
        
    /* Track unhandled interrupts */
    if (action_ret == IRQ_NONE) {
        desc->irqs_unhandled++;
        if (time_after(jiffies, desc->last_unhandled + HZ/10))
            desc->irqs_unhandled = 1;
        else if (desc->irqs_unhandled > 99900) {
            /* Disable spurious interrupt */
            __irq_disable(desc, true);
            desc->istate |= IRQS_SPURIOUS_DISABLED;
        }
        desc->last_unhandled = jiffies;
    }
}
```

### Power Management Integration

#### Suspend/Resume Infrastructure
```c
void suspend_device_irqs(void)
{
    struct irq_desc *desc;
    int irq;
    
    for_each_irq_desc(irq, desc) {
        if (irq_settings_is_nested_thread(desc))
            continue;
            
        raw_spin_lock_irqsave(&desc->lock, flags);
        sync = suspend_device_irq(desc);
        raw_spin_unlock_irqrestore(&desc->lock, flags);
        
        if (sync)
            synchronize_irq(irq);
    }
}
```

#### Wakeup IRQ Handling
```c
static bool suspend_device_irq(struct irq_desc *desc)
{
    struct irq_data *irqd = &desc->irq_data;
    
    if (irqd_is_wakeup_set(irqd)) {
        irqd_set(irqd, IRQD_WAKEUP_ARMED);
        
        if ((irqd_get_chip_flags(irqd) & IRQCHIP_ENABLE_WAKEUP_ON_SUSPEND) &&
            irqd_irq_disabled(irqd)) {
            __enable_irq(desc);
            irqd_set(irqd, IRQD_IRQ_ENABLED_ON_SUSPEND);
        }
        return true;
    }
    
    desc->istate |= IRQS_SUSPENDED;
    __disable_irq(desc);
    return true;
}
```

### Performance Optimizations

#### Cache-Friendly Design
```c
/* Cache line aligned IRQ descriptors */
struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp;

/* NUMA-aware allocation */
static struct irq_desc *alloc_desc(int irq, int node, unsigned int flags,
                                   const struct cpumask *affinity,
                                   struct module *owner)
{
    desc = kzalloc_node(sizeof(*desc), GFP_KERNEL, node);
    desc->kstat_irqs = alloc_percpu(struct irqstat);
    alloc_masks(desc, node);
    return desc;
}
```

#### Lock-Free Optimizations
```c
/* RCU-protected IRQ descriptor lookup */
#define for_each_irq_desc(irq, desc)                                   \
    for (irq = 0, desc = irq_to_desc(irq); irq < nr_irqs;             \
         irq++, desc = irq_to_desc(irq))                               \
        if (!desc)                                                     \
            ;                                                          \
        else

/* Lockless statistics updates */
#define __this_cpu_inc(pcp) __this_cpu_add(pcp, 1)
```

#### Maple Tree Optimizations
```c
static int irq_find_free_area(unsigned int from, unsigned int cnt)
{
    MA_STATE(mas, &sparse_irqs, 0, 0);
    
    if (mas_empty_area(&mas, from, MAX_SPARSE_IRQS, cnt))
        return -ENOSPC;
    return mas.index;
}
```

## Integration Points

### Hardware Abstraction Layer Integration
- **IRQ Domains**: Hardware IRQ number to Linux IRQ number mapping
- **IRQ Chips**: Hardware-specific interrupt controller operations
- **Hierarchical Controllers**: Support for complex interrupt routing (GIC, APIC)
- **MSI/MSI-X Support**: Message-signaled interrupt management

### Device Driver Integration
- **Request/Free API**: `request_irq()`, `free_irq()` interface
- **Threaded Interrupts**: Support for threaded interrupt handlers
- **Shared Interrupts**: Multiple devices sharing interrupt lines
- **Per-CPU Interrupts**: Timer and IPI interrupt support

### System Monitoring Integration
- **Procfs**: `/proc/interrupts` statistics display
- **Sysfs**: `/sys/kernel/irq/N/` configuration interface
- **Debugfs**: Comprehensive debugging information
- **Tracing**: Integration with ftrace and perf subsystems

### Power Management Integration
- **CPU Hotplug**: IRQ migration during CPU offline/online
- **System Suspend**: Selective IRQ suspension and wakeup support
- **Runtime PM**: Integration with device runtime power management
- **Energy Aware**: IRQ affinity considerations for power efficiency

This comprehensive IRQ descriptor infrastructure provides Linux with scalable, high-performance interrupt management capable of handling everything from simple embedded systems to large NUMA servers with thousands of interrupt sources, while maintaining microsecond-level latencies and comprehensive monitoring capabilities.