# Linux Kernel Reboot and Power Management (`kernel/reboot.c`)

## Overview

The `kernel/reboot.c` file implements the Linux kernel's system shutdown, reboot, and power management infrastructure. This critical component handles orderly system termination, emergency restart scenarios, and provides the interface for system administrators and programs to safely restart or power off the system. The implementation coordinates with hardware, drivers, and user-space to ensure data integrity and proper system state transitions.

## Core Architecture

### 1. System State Management

**Reboot Modes** - Lines 35-37:
```c
enum reboot_mode reboot_mode DEFAULT_REBOOT_MODE;  // Current reboot mode
enum reboot_mode panic_reboot_mode = REBOOT_UNDEFINED;  // Panic-specific mode
```

**Reboot Types** - Line 50:
- `BOOT_ACPI` - ACPI-based restart (default)
- `BOOT_BIOS` - BIOS-based restart
- `BOOT_EFI` - EFI-based restart
- `BOOT_KBD` - Keyboard controller restart
- `BOOT_TRIPLE` - Triple fault restart
- `BOOT_CF9_FORCE` - PCI configuration space restart

**System State Variables**:
- `reboot_cpu` - Specific CPU to perform restart on
- `reboot_force` - Force restart without normal shutdown
- `reboot_default` - Track if default settings are in use

### 2. Emergency Restart Infrastructure

**`emergency_restart()`** - Lines 92-98:
- **Purpose**: Immediate system restart without normal shutdown procedures
- **Safety**: Can be called from interrupt context
- **Operations**:
  - Dumps kernel messages (`KMSG_DUMP_EMERG`)
  - Sets system state to `SYSTEM_RESTART`
  - Calls architecture-specific emergency restart

**Hardware Integration**:
- Direct hardware manipulation for immediate restart
- Bypasses normal driver shutdown procedures
- Used during kernel panics and critical failures

### 3. Orderly Shutdown and Restart

**`kernel_restart_prepare()`** - Lines 100-106:
- **Notification Chain**: Calls reboot notifiers to prepare subsystems
- **State Transition**: Sets `SYSTEM_RESTART` state
- **User-mode Helpers**: Disables user-mode helper execution
- **Device Shutdown**: Initiates orderly device shutdown sequence

**`kernel_power_off()`** - Lines 705-716:
- **Complete Shutdown**: Coordinates full system power-off
- **System Preparation**: Calls shutdown preparation routines
- **CPU Migration**: Migrates to designated reboot CPU
- **Core Shutdown**: Executes syscore shutdown operations

### 4. Notifier Chain System

**Reboot Notifiers** - Lines 82, 108-137:
- **`register_reboot_notifier()`**: Register shutdown notification callbacks
- **`unregister_reboot_notifier()`**: Remove notification callbacks
- **`devm_register_reboot_notifier()`**: Device-managed notifier registration
- **Purpose**: Allows subsystems to perform cleanup before restart/shutdown

**Restart Handlers** - Lines 170-243:
- **Priority-based System**: Handlers registered with priority levels
  - `255`: Highest priority (watchdog systems)
  - `128`: Default priority (standard restart mechanisms)  
  - `0`: Last resort (limited capability handlers)
- **`register_restart_handler()`**: Register hardware-specific restart methods

## System Call Interface

### 1. Reboot System Call

**`sys_reboot()`** - Lines 728-816:
- **Security**: Requires `CAP_SYS_BOOT` capability
- **Magic Numbers**: Uses magic number validation for safety
- **PID Namespace Aware**: Handles containerized environments

**Supported Commands**:
- `LINUX_REBOOT_CMD_RESTART` - Standard system restart
- `LINUX_REBOOT_CMD_RESTART2` - Restart with custom command string
- `LINUX_REBOOT_CMD_HALT` - Halt system without power-off
- `LINUX_REBOOT_CMD_POWER_OFF` - Power off the system
- `LINUX_REBOOT_CMD_CAD_ON/OFF` - Enable/disable Ctrl-Alt-Del
- `LINUX_REBOOT_CMD_KEXEC` - Execute kexec kernel switch
- `LINUX_REBOOT_CMD_SW_SUSPEND` - Hibernate system

### 2. Magic Number Validation

**Security Protection** - Lines 740-745:
```c
if (magic1 != LINUX_REBOOT_MAGIC1 ||
    (magic2 != LINUX_REBOOT_MAGIC2 &&
     magic2 != LINUX_REBOOT_MAGIC2A &&
     magic2 != LINUX_REBOOT_MAGIC2B &&
     magic2 != LINUX_REBOOT_MAGIC2C))
    return -EINVAL;
```

**Purpose**: Prevents accidental system restarts from programming errors

## Ctrl-Alt-Del Handling

### 1. Keyboard Interrupt Handler

**`ctrl_alt_del()`** - Lines 828-836:
- **Deferred Execution**: Uses work queue for safe execution
- **Configurable Behavior**: Controlled by `C_A_D` variable
- **Options**:
  - Immediate restart when enabled
  - Send SIGINT to configured process when disabled

**`deferred_cad()`** - Lines 818-821:
- **Work Queue Handler**: Safely executes restart from process context
- **Simple Implementation**: Direct call to `kernel_restart(NULL)`

### 2. CAD Process Management

**`cad_pid`** - Lines 27-28:
- **Custom Handler**: Allows designation of specific process for Ctrl-Alt-Del
- **Signal Delivery**: Sends SIGINT to designated process instead of restarting
- **Configuration**: Managed through sysctl interface

## Orderly vs. Forced Operations

### 1. Orderly Shutdown

**`__orderly_poweroff()`** - Lines 877-896:
- **User-space Command**: Executes `/sbin/poweroff` command
- **Fallback Mechanism**: Forces kernel power-off if user-space fails
- **Emergency Sync**: Syncs filesystems before forced shutdown

**`__orderly_reboot()`** - Lines 862-875:
- **User-space Command**: Executes `/sbin/reboot` command
- **Emergency Handling**: Forces kernel restart if user-space fails
- **Data Protection**: Emergency sync before forced restart

### 2. Emergency Operations

**`emergency_sync()`** - Called during forced operations:
- **Filesystem Sync**: Attempts to sync dirty data to storage
- **Best Effort**: May not complete if system is severely compromised
- **Data Protection**: Minimizes data loss during emergency shutdown

## Hardware Protection Integration

### 1. Hardware Protection Action

**`hw_protection_action`** - Line 39:
- **`HWPROT_ACT_SHUTDOWN`**: Shutdown system on hardware protection trigger
- **Integration**: Coordinates with thermal and hardware monitoring
- **Safety**: Prevents hardware damage from overheating or other issues

### 2. Power Management Integration

**Power-off Capabilities**:
- **`kernel_can_power_off()`**: Checks if system supports power-off
- **Fallback to Halt**: Automatically falls back to halt if power-off unavailable
- **`poweroff_fallback_to_halt`**: Tracks fallback state for user notification

## Configuration Interfaces

### 1. Sysfs Interface

**`/sys/kernel/reboot/` Attributes** - Lines 1355-1396:
- **`mode`**: Reboot mode (cold, warm, hard, soft, gpio)
- **`type`**: Reboot type (acpi, bios, efi, kbd, triple, pci) - x86 only
- **`force`**: Force reboot without normal shutdown - x86 only
- **`cpu`**: Specific CPU to perform reboot on - SMP only
- **`hw_protection`**: Hardware protection action

### 2. Sysctl Interface

**`/proc/sys/kernel/` Controls** - Lines 1369-1392:
- **`poweroff_cmd`**: Path to user-space poweroff command
- **`ctrl-alt-del`**: Enable/disable Ctrl-Alt-Del restart behavior

## System Transition Coordination

### 1. System Transition Mutex

**`system_transition_mutex`** - Line 718:
- **Serialization**: Ensures only one system transition at a time
- **Race Prevention**: Prevents concurrent reboot/shutdown operations
- **Critical Section**: Protects system state changes

### 2. Migration to Reboot CPU

**`migrate_to_reboot_cpu()`**:
- **CPU Affinity**: Ensures restart operations run on designated CPU
- **Hardware Requirements**: Some hardware requires specific CPU for restart
- **SMP Coordination**: Manages multi-processor restart coordination

## Architecture Integration

### 1. Platform-Specific Operations

**Machine Interface**:
- `machine_restart()` - Platform-specific restart implementation
- `machine_halt()` - Platform-specific halt implementation  
- `machine_power_off()` - Platform-specific power-off implementation
- `machine_emergency_restart()` - Emergency restart without coordination

### 2. Syscore Operations

**`syscore_shutdown()`** - Line 710:
- **Core System Shutdown**: Shuts down critical system components
- **IRQ Disabled Context**: Operates with interrupts disabled
- **Final Cleanup**: Last-chance cleanup before hardware reset

## Error Handling and Robustness

### 1. Fallback Mechanisms

**Multi-level Fallback**:
1. User-space orderly shutdown/restart
2. Kernel-driven shutdown/restart  
3. Emergency restart mechanisms
4. Hardware-level reset

### 2. Failure Recovery

**Command Execution Failures**:
- **User-space Timeout**: Automatic fallback to kernel operations
- **Resource Exhaustion**: Emergency sync and immediate action
- **Hardware Failures**: Multiple restart method attempts

## Security Considerations

### 1. Capability Requirements

**Access Control**:
- `CAP_SYS_BOOT` required for all reboot operations
- Namespace-aware capability checking
- Configuration changes require administrative privileges

### 2. Magic Number Protection

**Accidental Restart Prevention**:
- Multiple magic number validation
- Prevents library bugs from causing restarts
- Historical Linus Torvalds design for safety

## Integration with Container Systems

### 1. PID Namespace Support

**`reboot_pid_ns()`** - Line 752:
- **Container Isolation**: Handles reboot in containerized environments
- **Process Termination**: May terminate container instead of system reboot
- **Namespace Awareness**: Respects container boundaries

### 2. User Namespace Integration

**Capability Checking**:
- Uses namespace-aware capability functions
- Prevents container escape through reboot operations
- Maintains security boundaries

This reboot system represents a critical kernel infrastructure that balances safety, reliability, and flexibility. It provides multiple layers of protection against accidental restarts while ensuring that legitimate shutdown and restart operations can proceed even under adverse conditions, making it essential for system administration and emergency recovery scenarios.