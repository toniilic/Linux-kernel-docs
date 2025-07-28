# Linux Kernel Panic Implementation (`kernel/panic.c`)

## Overview

The `kernel/panic.c` file is the core implementation of Linux kernel panic handling, a critical safety mechanism that brings the system to a controlled halt when unrecoverable errors occur. This file contains the fundamental error handling infrastructure that ensures system reliability and provides debugging information during critical failures.

## Key Components

### 1. Core Panic Function

**`panic(const char *fmt, ...)`** - Lines 315-507
- **Purpose**: The primary kernel panic function that never returns
- **Behavior**: 
  - Disables interrupts and preemption to prevent further damage
  - Uses atomic operations to ensure only one CPU handles the panic
  - Formats and displays the panic message
  - Coordinates system shutdown procedures
  - Handles crash kernel execution (kexec) if configured
  - Provides configurable timeout before reboot

**Key Features**:
- **CPU Coordination**: Uses `panic_cpu` atomic variable to ensure single-CPU panic handling
- **Message Formatting**: Creates formatted panic messages with `vscnprintf()`
- **System State Preservation**: Calls `console_verbose()` and `bust_spinlocks(1)`
- **Debug Integration**: Supports KGDB debugging during panic

### 2. NMI Panic Handling

**`nmi_panic(struct pt_regs *regs, const char *msg)`** - Lines 226-238
- **Purpose**: Specialized panic handler for Non-Maskable Interrupt (NMI) contexts
- **Safety**: Prevents recursive panics and ensures proper CPU coordination
- **Architecture Support**: Provides hooks for architecture-specific crash handling

### 3. Configuration Variables

**Runtime Behavior Control**:
- `panic_timeout` - Seconds to wait before reboot (0 = infinite wait)
- `panic_on_oops` - Whether to panic on oops conditions
- `panic_on_warn` - Whether to panic on warning conditions
- `panic_print` - Bitmask controlling what information to display during panic

**Information Display Flags** (Lines 72-80):
```c
#define PANIC_PRINT_TASK_INFO      0x00000001  // Show task information
#define PANIC_PRINT_MEM_INFO       0x00000002  // Show memory information  
#define PANIC_PRINT_TIMER_INFO     0x00000004  // Show timer information
#define PANIC_PRINT_LOCK_INFO      0x00000008  // Show lock debugging info
#define PANIC_PRINT_FTRACE_INFO    0x00000010  // Show function trace
#define PANIC_PRINT_ALL_PRINTK_MSG 0x00000020  // Show all printk messages
#define PANIC_PRINT_ALL_CPU_BT     0x00000040  // Show all CPU backtraces
#define PANIC_PRINT_BLOCKED_TASKS  0x00000080  // Show blocked tasks
```

### 4. System Control Integration

**Sysctl Interface** - Lines 87-143:
- Provides `/proc/sys/kernel/` interfaces for runtime configuration
- Controls panic behavior, timeouts, and information display
- Allows system administrators to tune panic behavior

**Sysfs Integration** - Lines 148-163:
- Exposes warn count through `/sys/kernel/warn_count`
- Provides read-only access to warning statistics

### 5. CPU Shutdown Coordination

**`panic_other_cpus_shutdown(bool crash_kexec)`** - Lines 286-307
- **Purpose**: Coordinates shutdown of secondary CPUs during panic
- **Backtrace Support**: Triggers all-CPU backtraces if configured
- **Crash Kernel Support**: Uses appropriate CPU stopping mechanism based on crash kernel usage

**CPU Stop Functions**:
- `panic_smp_self_stop()` - Default CPU self-stop implementation
- `crash_smp_send_stop()` - Crash kernel aware CPU stopping
- `nmi_panic_self_stop()` - NMI context CPU stopping

### 6. Kernel Taint Tracking

**Taint Management** - Lines 522-641:
- **Purpose**: Tracks kernel integrity violations that may affect reliability
- **Taint Flags**: Comprehensive set of flags indicating various integrity issues
- **Panic Integration**: Can trigger panic when specific taint conditions occur

**Major Taint Categories**:
- `TAINT_PROPRIETARY_MODULE` ('P') - Proprietary module loaded
- `TAINT_FORCED_MODULE` ('F') - Module force-loaded
- `TAINT_CPU_OUT_OF_SPEC` ('S') - CPU out of specification
- `TAINT_MACHINE_CHECK` ('M') - Machine check error
- `TAINT_BAD_PAGE` ('B') - Bad page detected
- `TAINT_USER` ('U') - User-requested taint
- `TAINT_DIE` ('D') - System died
- `TAINT_WARN` ('W') - Warning issued

### 7. OOPS Handling Infrastructure

**`oops_enter()` and `oops_exit()`** - Lines 715-742:
- **Purpose**: Coordinate OOPS (kernel error) handling across CPUs
- **Synchronization**: Ensures clean OOPS display without interference
- **Debugging Integration**: Disables locks debugging and enables tracing

**`__warn()` Function** - Lines 749-791:
- **Purpose**: Core warning implementation
- **Information Display**: Shows CPU, PID, location, and stack trace
- **Taint Integration**: Adds appropriate taint flags
- **Panic Integration**: Can trigger panic based on configuration

### 8. Stack Protection

**`__stack_chk_fail()`** - Lines 862-875:
- **Purpose**: Handler for GCC stack protector failures
- **Security**: Detects stack buffer overflows
- **Response**: Immediately panics the system to prevent exploitation

## Critical Design Patterns

### 1. Atomic CPU Coordination
The panic system uses atomic compare-and-swap operations to ensure only one CPU handles the panic, preventing chaotic multi-CPU panic scenarios.

### 2. Graceful Degradation
The code is designed to work even when the kernel is in a corrupted state, using minimal kernel services and defensive programming.

### 3. Configurable Behavior
Extensive runtime configuration allows system administrators to tailor panic behavior to their specific needs (debugging vs. production).

### 4. Information Preservation
The panic handler preserves and displays maximum diagnostic information while maintaining system stability during the shutdown process.

## Security Considerations

1. **Stack Protection Integration**: Immediate panic on stack corruption prevents potential exploits
2. **Taint Tracking**: Helps identify potentially compromised kernel state
3. **NMI Safety**: Proper handling of non-maskable interrupts prevents security bypasses
4. **Debug Information Control**: Configurable information display prevents information leakage

## Architecture Integration

The panic system provides weak function implementations that can be overridden by architecture-specific code:
- `panic_smp_self_stop()` - Architecture-specific CPU stopping
- `crash_smp_send_stop()` - Architecture-specific crash handling
- Platform-specific panic blinking and reboot mechanisms

## Usage Patterns

### Emergency Situations
- Memory corruption detection
- Hardware failures (machine check errors)
- Security violations (stack protection failures)
- Unrecoverable software errors

### Development and Debugging
- Configurable information display for debugging
- Integration with crash dump mechanisms
- Support for kernel debuggers (KGDB)

This panic implementation represents a critical kernel safety mechanism that balances the need for system protection with diagnostic information preservation, making it an essential component of kernel reliability and security.