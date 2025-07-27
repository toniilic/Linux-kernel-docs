# init/main.c - Linux Kernel Bootstrap and Initialization

## Overview

This file implements the core kernel bootstrap sequence for Linux, originally developed by Linus Torvalds in 1991-1992. It contains the fundamental initialization code that brings the Linux system from a minimal boot state to a fully operational kernel capable of running user-space processes. The file orchestrates the complex sequence of subsystem initialization, process creation, and the crucial transition from kernel space to user space.

## Historical Development

### Key Contributors and Milestones
- **Linus Torvalds (1991-1992)**: Original kernel initialization framework
- **Gadi Oxman (1995)**: NFS root filesystem support
- **Werner Almesberger & Hans Lermen (1996)**: initrd and change_root support
- **Paul Gortmaker (1996)**: Early compiler validation
- **Michael A. Griffith**: Simplified init process starting

### Evolution Timeline
- **1991-1992**: Basic kernel initialization sequence
- **1995**: Network root filesystem support
- **1996**: Initial RAM disk support and root filesystem switching
- **2000s**: SMP initialization and scalability improvements
- **2010s**: Modern security features, container support, and performance optimizations
- **2020s**: Advanced tracing, boot configuration, and security hardening

### Design Philosophy
The initialization system is built around the principle of ordered dependency resolution, ensuring that kernel subsystems are initialized in the correct sequence while providing flexibility for different boot scenarios and hardware configurations.

## Core Concepts

### Boot Sequence Architecture

#### Kernel Boot Pipeline
```
Bootloader → start_kernel() → rest_init() → kernel_init() → User Space
     ↓           ↓              ↓             ↓              ↓
[Hardware]  [Core Init]   [Process Setup] [Final Init]  [/sbin/init]
```

#### System State Transitions
```
SYSTEM_BOOTING → SYSTEM_SCHEDULING → SYSTEM_RUNNING → SYSTEM_HALT/SUSPEND
       ↓               ↓                    ↓                ↓
[Early Boot]    [Process Creation]   [Normal Operation] [Shutdown/Sleep]
```

#### Initialization Phases
- **Early Boot**: Basic CPU, memory, and interrupt setup
- **Subsystem Init**: Major kernel subsystems initialization
- **Driver Init**: Device drivers and hardware initialization
- **User Transition**: Preparation for and transition to user space

## Key Data Structures

### System State Management
```c
enum system_states {
    SYSTEM_BOOTING,         /* Early boot phase */
    SYSTEM_SCHEDULING,      /* Process scheduling enabled */
    SYSTEM_RUNNING,         /* Normal operation */
    SYSTEM_HALT,           /* System halting */
    SYSTEM_POWER_OFF,      /* Power off */
    SYSTEM_RESTART,        /* System restart */
    SYSTEM_SUSPEND,        /* System suspend */
    SYSTEM_HIBERNATE,      /* System hibernation */
    SYSTEM_FREEING_INITMEM, /* Freeing init memory */
};

extern enum system_states system_state;
```

### Command Line Processing
```c
/* Boot command line handling */
#define MAX_INIT_ARGS CONFIG_INIT_ENV_ARG_LIMIT
#define MAX_INIT_ENVS CONFIG_INIT_ENV_ARG_LIMIT

char __initdata boot_command_line[COMMAND_LINE_SIZE];  /* Untouched from arch */
char *saved_command_line __ro_after_init;             /* Saved for /proc */
static char *static_command_line;                     /* For parameter parsing */
static char *extra_command_line;                      /* Extra parameters */
static char *extra_init_args;                         /* Extra init arguments */

static const char *argv_init[MAX_INIT_ARGS+2] = { "init", NULL, };
const char *envp_init[MAX_INIT_ENVS+2] = { "HOME=/", "TERM=linux", NULL, };
```

### Init Call System
```c
typedef int (*initcall_t)(void);
typedef initcall_t initcall_entry_t;

/* Initcall levels for ordered initialization */
static const char *initcall_level_names[] __initdata = {
    "pure",      /* pure_initcall() - Level 0 */
    "core",      /* core_initcall() - Level 1 */
    "postcore",  /* postcore_initcall() - Level 2 */
    "arch",      /* arch_initcall() - Level 3 */
    "subsys",    /* subsys_initcall() - Level 4 */
    "fs",        /* fs_initcall() - Level 5 */
    "device",    /* device_initcall() - Level 6 */
    "late",      /* late_initcall() - Level 7 */
};

static initcall_entry_t *initcall_levels[] __initdata = {
    __initcall0_start, __initcall1_start, __initcall2_start,
    __initcall3_start, __initcall4_start, __initcall5_start,
    __initcall6_start, __initcall7_start, __initcall_end,
};
```

## Core Functions

### Primary Entry Point

#### `start_kernel()` - Main Kernel Entry Point
```c
void start_kernel(void)
```

**Purpose**: Primary kernel initialization function called by architecture-specific boot code

**Initialization Sequence**:
1. **Early Setup Phase**:
   ```c
   set_task_stack_end_magic(&init_task);  /* Stack overflow protection */
   smp_setup_processor_id();             /* Processor identification */
   debug_objects_early_init();           /* Debug object tracking */
   init_vmlinux_build_id();             /* Build ID setup */
   cgroup_init_early();                 /* Early cgroup setup */
   local_irq_disable();                 /* Disable interrupts */
   early_boot_irqs_disabled = true;     /* Mark early boot state */
   ```

2. **Architecture and Hardware Setup**:
   ```c
   boot_cpu_init();                     /* Boot CPU initialization */
   page_address_init();                 /* Page address mapping */
   pr_notice("%s", linux_banner);       /* Display kernel banner */
   setup_arch(&command_line);           /* Architecture-specific setup */
   jump_label_init();                   /* Static key infrastructure */
   static_call_init();                  /* Static call infrastructure */
   early_security_init();               /* Early security framework */
   ```

3. **Command Line and Configuration**:
   ```c
   setup_boot_config();                 /* Boot configuration */
   setup_command_line(command_line);    /* Command line processing */
   setup_nr_cpu_ids();                 /* CPU count setup */
   setup_per_cpu_areas();              /* Per-CPU memory areas */
   parse_early_param();                /* Early parameter parsing */
   ```

4. **Memory and Core Subsystems**:
   ```c
   mm_core_init();                      /* Memory management core */
   sched_init();                        /* Scheduler initialization */
   radix_tree_init();                   /* Radix tree infrastructure */
   maple_tree_init();                   /* Maple tree infrastructure */
   workqueue_init_early();              /* Early workqueue setup */
   rcu_init();                          /* RCU subsystem */
   ```

5. **Finalization and Transition**:
   ```c
   early_irq_init();                    /* Interrupt infrastructure */
   init_timers();                       /* Timer subsystem */
   srcu_init();                         /* SRCU initialization */
   hrtimers_init();                     /* High-resolution timers */
   softirq_init();                      /* Softirq infrastructure */
   time_init();                         /* Time subsystem */
   rest_init();                         /* Transition to process model */
   ```

### Process Creation and Transition

#### `rest_init()` - Process Model Initialization
```c
static noinline void __ref rest_init(void)
```

**Purpose**: Create fundamental processes and transition to process-based execution

**Process Creation Sequence**:
1. **Init Process Creation (PID 1)**:
   ```c
   pid = user_mode_thread(kernel_init, NULL, CLONE_FS);
   rcu_read_lock();
   tsk = find_task_by_pid_ns(pid, &init_pid_ns);
   tsk->flags |= PF_NO_SETAFFINITY;
   set_cpus_allowed_ptr(tsk, cpumask_of(smp_processor_id()));
   rcu_read_unlock();
   ```

2. **Kthread Daemon Creation (PID 2)**:
   ```c
   pid = kernel_thread(kthreadd, NULL, NULL, CLONE_FS | CLONE_FILES);
   rcu_read_lock();
   kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
   rcu_read_unlock();
   complete(&kthreadd_done);
   ```

3. **System State Transition**:
   ```c
   system_state = SYSTEM_SCHEDULING;    /* Enable scheduling features */
   schedule_preempt_disabled();        /* Force initial schedule */
   cpu_startup_entry(CPUHP_ONLINE);    /* Enter idle loop */
   ```

#### `kernel_init()` - Init Process Main Function
```c
static int __ref kernel_init(void *unused)
```

**Purpose**: Main function for PID 1, handles transition to user space

**Initialization Process**:
1. **Wait for Infrastructure**:
   ```c
   wait_for_completion(&kthreadd_done); /* Wait for kthreadd ready */
   ```

2. **System Initialization**:
   ```c
   kernel_init_freeable();             /* Complete kernel initialization */
   ```

3. **Memory Cleanup Phase**:
   ```c
   async_synchronize_full();           /* Wait for async operations */
   system_state = SYSTEM_FREEING_INITMEM;
   kprobe_free_init_mem();            /* Free probe init memory */
   ftrace_free_init_mem();            /* Free ftrace init memory */
   free_initmem();                    /* Free init memory sections */
   mark_readonly();                   /* Mark kernel text read-only */
   ```

4. **Final System Setup**:
   ```c
   pti_finalize();                    /* Page Table Isolation */
   system_state = SYSTEM_RUNNING;     /* Mark system as running */
   numa_default_policy();            /* NUMA policy setup */
   rcu_end_inkernel_boot();          /* End RCU boot phase */
   ```

5. **User Space Transition**:
   ```c
   /* Try various init programs in order */
   if (ramdisk_execute_command) {
       ret = run_init_process(ramdisk_execute_command);
   }
   if (execute_command) {
       ret = run_init_process(execute_command);
   }
   if (CONFIG_DEFAULT_INIT[0] != '\0') {
       ret = run_init_process(CONFIG_DEFAULT_INIT);
   }
   ret = run_init_process("/sbin/init");
   ret = run_init_process("/etc/init");
   ret = run_init_process("/bin/init");
   ret = run_init_process("/bin/sh");
   panic("No working init found.");
   ```

### Init Call System

#### `do_initcalls()` - Execute All Initialization Functions
```c
static void __init do_initcalls(void)
```

**Purpose**: Execute all registered initialization functions in priority order

**Execution Process**:
```c
for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++) {
    do_initcall_level(level, command_line);
}
```

#### `do_initcall_level()` - Execute Single Priority Level
```c
static void __init do_initcall_level(int level, char *command_line)
```

**Level Processing**:
1. **Parameter Parsing**: Parse level-specific command line parameters
2. **Function Iteration**: Execute all functions registered at this level
3. **Debugging**: Provide detailed timing and tracing information

#### `do_one_initcall()` - Execute Single Initialization Function
```c
int __init_or_module do_one_initcall(initcall_t fn)
```

**Execution Safety**:
```c
/* Timing and debugging */
if (initcall_debug)
    ret = do_one_initcall_debug(fn);
else
    ret = fn();

/* Validate system state after call */
if (preempt_count() != count) {
    sprintf(msgbuf, "preemption imbalance ");
    preempt_count_set(count);
}
if (irqs_disabled()) {
    sprintf(msgbuf, "disabled interrupts ");
    local_irq_enable();
}
```

### Priority Levels and Subsystem Initialization

#### Initcall Priority Levels
1. **pure_initcall()** (Level 0): No dependencies, static initialization
2. **core_initcall()** (Level 1): Core kernel subsystems
3. **postcore_initcall()** (Level 2): Post-core infrastructure
4. **arch_initcall()** (Level 3): Architecture-specific initialization
5. **subsys_initcall()** (Level 4): Major kernel subsystems
6. **fs_initcall()** (Level 5): Filesystem registration
7. **device_initcall()** (Level 6): Device drivers
8. **late_initcall()** (Level 7): Late initialization

#### Example Subsystem Registrations
```c
/* Core memory management */
core_initcall(mm_sysfs_init);

/* Virtual terminal console */
postcore_initcall(vtconsole_class_init);

/* IPC subsystem */
device_initcall(ipc_init);

/* Message queue filesystem */
device_initcall(init_mqueue_fs);
```

### Command Line Processing

#### Parameter Parsing Infrastructure
```c
static int __init unknown_bootoption(char *param, char *val,
                                   const char *unused, void *arg)
static bool __init obsolete_checksetup(char *line)
```

**Features**:
- **Early Parameters**: Processed before main kernel initialization
- **Module Parameters**: Passed to loadable modules
- **Obsolete Handling**: Support for legacy parameter formats
- **Unknown Parameter**: Graceful handling of unrecognized parameters

#### Boot Configuration Support
```c
#ifdef CONFIG_BOOT_CONFIG
static bool bootconfig_found;
static size_t initargs_offs;
#endif
```

**Boot Config Features**:
- **Structured Configuration**: Hierarchical boot parameter organization
- **Extended Syntax**: Advanced parameter specification
- **Dynamic Loading**: Runtime boot configuration modification
- **Validation**: Configuration syntax and semantic validation

## Advanced Features

### Security and Hardening

#### Stack Protection
```c
set_task_stack_end_magic(&init_task);  /* Stack overflow detection */
```

#### Memory Protection
```c
mark_readonly();                       /* Mark kernel text read-only */
pti_finalize();                       /* Page Table Isolation */
```

#### Early Security Framework
```c
early_security_init();                /* LSM early initialization */
```

### Performance and Scalability

#### SMP Initialization
```c
smp_prepare_boot_cpu();               /* Boot CPU preparation */
early_numa_node_init();               /* NUMA topology */
boot_cpu_hotplug_init();             /* CPU hotplug infrastructure */
```

#### Per-CPU Infrastructure
```c
setup_per_cpu_areas();               /* Per-CPU memory allocation */
```

#### Lock-Free Infrastructure
```c
jump_label_init();                    /* Static branch optimization */
static_call_init();                   /* Static call optimization */
```

### Debugging and Tracing

#### Debug Infrastructure
```c
debug_objects_early_init();           /* Object lifecycle tracking */
early_trace_init();                   /* Early tracing setup */
```

#### Initcall Debugging
```c
static bool initcall_debug;           /* Detailed initcall tracing */
```

**Debug Features**:
- **Timing Information**: Precise timing for each initcall
- **State Validation**: Check system state after each call
- **Error Detection**: Detect and report initialization failures
- **Blacklisting**: Disable problematic initialization functions

### Boot Configuration and Flexibility

#### Multiple Init Support
```c
static char *execute_command;         /* Custom init command */
static char *ramdisk_execute_command = "/init"; /* Ramdisk init */
```

#### Emergency Recovery
```c
/* Fallback init locations */
"/sbin/init", "/etc/init", "/bin/init", "/bin/sh"
```

#### Reset and Recovery
```c
unsigned int reset_devices;           /* Force device reset */
EXPORT_SYMBOL(reset_devices);
```

## Integration Points

### Architecture Integration
- **Platform Setup**: `setup_arch()` for platform-specific initialization
- **Boot Memory**: Integration with bootmem and memblock allocators
- **Interrupt Setup**: Architecture-specific interrupt initialization
- **Timer Setup**: Platform timer and clocksource initialization

### Memory Management Integration
- **Early Allocators**: Bootmem and memblock integration
- **Page Allocator**: Transition to buddy allocator
- **Virtual Memory**: MMU setup and virtual memory initialization
- **NUMA Support**: NUMA topology discovery and setup

### Process Management Integration
- **Task Structure**: Initialize root task (swapper/init_task)
- **Scheduler**: Process scheduler initialization
- **Thread Creation**: Kernel and user thread infrastructure
- **PID Management**: Process ID namespace setup

### Device and Driver Integration
- **Device Model**: Device driver framework initialization
- **Bus Systems**: Platform, PCI, USB bus initialization
- **Driver Loading**: Module loading and driver binding
- **Firmware Interface**: ACPI, device tree, and firmware integration

This comprehensive kernel initialization implementation provides the foundation for bringing Linux from bare hardware to a fully operational system, ensuring proper subsystem ordering, security, and compatibility across diverse hardware platforms and deployment scenarios.