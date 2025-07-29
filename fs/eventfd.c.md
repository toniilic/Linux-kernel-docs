# eventfd.c - Event File Descriptor Implementation

## Overview

The `eventfd.c` file implements event file descriptors, a Linux-specific mechanism for event notification between kernel and userspace, or between different parts of the kernel. Eventfd provides a simple counter-based signaling mechanism that integrates seamlessly with standard I/O multiplexing interfaces like poll/select/epoll.

## File Location
- **Path**: `fs/eventfd.c`
- **License**: GPL-2.0-only
- **Author**: Davide Libenzi (davidel@xmailserver.org) - 2007

## Core Concepts

### Event File Descriptor Purpose
- **Lightweight Signaling**: Provides efficient event notification mechanism
- **I/O Integration**: Works with poll/select/epoll for event-driven programming
- **Counter-Based**: Uses 64-bit counter for event accumulation
- **Kernel-Userspace Bridge**: Enables kernel components to signal userspace efficiently

### Key Characteristics
- **Non-Blocking Operations**: Supports both blocking and non-blocking I/O
- **Semaphore Mode**: Optional semaphore-like behavior with EFD_SEMAPHORE flag
- **Overflow Detection**: Handles counter overflow gracefully
- **Thread-Safe**: Provides proper synchronization for concurrent access

## Core Data Structures

### eventfd_ctx
```c
struct eventfd_ctx {
    struct kref kref;           // Reference counting
    wait_queue_head_t wqh;      // Wait queue for blocking operations
    __u64 count;                // Event counter
    unsigned int flags;         // Behavioral flags (EFD_SEMAPHORE, etc.)
    int id;                     // Unique identifier
};
```

The main context structure representing an eventfd instance.

### ID Management
```c
static DEFINE_IDA(eventfd_ida);
```
IDA (ID Allocator) for assigning unique identifiers to eventfd instances.

## Core Functions

### Signal Operations

#### `eventfd_signal_mask(struct eventfd_ctx *ctx, __poll_t mask)`
Increments the event counter and wakes up waiting processes.

**Parameters**:
- `ctx`: Eventfd context
- `mask`: Additional poll mask bits

**Key Features**:
- **Overflow Handling**: Prevents counter from exceeding ULLONG_MAX
- **Deadlock Prevention**: Uses `current->in_eventfd` flag to prevent recursion
- **Atomic Operation**: Uses spinlock for thread-safe counter increment
- **Efficient Wakeup**: Only wakes up if waitqueue is active

**Deadlock Prevention**:
```c
if (WARN_ON_ONCE(current->in_eventfd))
    return;
```
Prevents recursive calls that could cause deadlock or stack overflow.

**Algorithm**:
1. Check for recursion and warn if detected
2. Acquire waitqueue spinlock with interrupts disabled
3. Set recursion flag
4. Increment counter if not at maximum
5. Wake up waiters if any exist
6. Clear recursion flag and release lock

### Reference Management

#### `eventfd_ctx_put(struct eventfd_ctx *ctx)`
Releases a reference to eventfd context.
- **Parameters**: Eventfd context
- **Implementation**: Uses `kref_put()` with `eventfd_free` callback
- **Thread Safety**: Reference counting ensures safe deallocation

#### `eventfd_free(struct kref *kref)`
Callback function for final reference release.
- **Parameters**: kref structure from eventfd_ctx
- **Action**: Calls `eventfd_free_ctx()` for cleanup

#### `eventfd_free_ctx(struct eventfd_ctx *ctx)`
Performs actual cleanup of eventfd context.
- **ID Cleanup**: Releases ID back to `eventfd_ida`
- **Memory**: Frees context structure
- **Conditional**: Only frees ID if valid (>= 0)

### File Operations

#### `eventfd_release(struct inode *inode, struct file *file)`
Called when file descriptor is closed.
- **Parameters**: Inode and file structures
- **Actions**:
  1. Wakes up all waiters with EPOLLHUP
  2. Releases context reference
- **Return**: Always returns 0

#### `eventfd_poll(struct file *file, poll_table *wait)`
Implements poll/select support for eventfd.

**Parameters**:
- `file`: File structure
- `wait`: Poll table for registering wait queue

**Returns**: Poll mask indicating current file state

**Memory Ordering Analysis**:
The function includes extensive comments about memory ordering and race conditions:

**Safe Race Condition**:
```c
// Poll thread                    Write thread
lock ctx->wqh.lock (poll_wait)   
count = ctx->count               
__add_wait_queue                 
unlock ctx->wqh.lock             
                                 lock ctx->wqh.lock
                                 ctx->count += n
                                 wake_up_locked_poll
                                 unlock ctx->wqh.lock
```

**Prevented Race Condition**:
```c
// This CANNOT happen due to memory barriers
count = ctx->count (INVALID!)    
                                 lock ctx->wqh.lock
                                 ctx->count += n
                                 // waitqueue_active is false
                                 // no wakeup occurs!
                                 unlock ctx->wqh.lock
lock ctx->wqh.lock (poll_wait)   
__add_wait_queue                 
unlock ctx->wqh.lock             
// Missed wakeup!
```

**Poll State Logic**:
- **EPOLLIN**: Set when count > 0 (data available to read)
- **EPOLLERR**: Set when count == ULLONG_MAX (overflow condition)
- **EPOLLOUT**: Set when count < ULLONG_MAX - 1 (space available to write)

### Read Operations

#### `eventfd_ctx_do_read(struct eventfd_ctx *ctx, __u64 *cnt)`
Performs the actual read operation on eventfd counter.

**Parameters**:
- `ctx`: Eventfd context
- `cnt`: Output pointer for counter value

**Locking**: Requires caller to hold `ctx->wqh.lock`

**Behavior**:
- **Normal Mode**: Returns and resets entire counter value
- **Semaphore Mode**: Returns 1 and decrements counter by 1 (if count > 0)

**Implementation**:
```c
*cnt = ((ctx->flags & EFD_SEMAPHORE) && ctx->count) ? 1 : ctx->count;
ctx->count -= *cnt;
```

#### `eventfd_ctx_remove_wait_queue(struct eventfd_ctx *ctx, wait_queue_entry_t *wait, __u64 *cnt)`
Atomically removes wait queue entry and reads counter.

**Parameters**:
- `ctx`: Eventfd context
- `wait`: Wait queue entry to remove
- `cnt`: Output pointer for counter value

**Returns**: 0 on success, -EAGAIN if would block

**Purpose**: Enables atomic "remove from waitqueue and read" operation

**Use Cases**:
- Cleanup during process termination
- Cancellation of pending operations
- Optimization to avoid separate remove and read operations

## Behavioral Modes

### Normal Mode
- **Read Behavior**: Returns entire counter value and resets to 0
- **Write Behavior**: Adds written value to counter
- **Use Case**: Event counting and accumulation

### Semaphore Mode (EFD_SEMAPHORE)
- **Read Behavior**: Returns 1 and decrements counter by 1
- **Write Behavior**: Adds written value to counter
- **Use Case**: Semaphore-like resource counting

## Overflow Handling

### Counter Overflow
- **Maximum Value**: ULLONG_MAX (2⁶⁴ - 1)
- **Overflow Detection**: Counter increment stops at maximum
- **Error Signaling**: EPOLLERR flag set when at maximum
- **Prevention**: Write operations check for overflow before incrementing

### Signal Overflow
When `eventfd_signal_mask()` is called and counter is at maximum:
1. Counter remains unchanged
2. Wakeup still occurs (important for error handling)
3. Poll returns EPOLLERR to indicate overflow condition

## Synchronization and Thread Safety

### Locking Strategy
- **Primary Lock**: `ctx->wqh.lock` protects counter and wait queue
- **Atomic Reads**: `READ_ONCE()` for lockless counter reads in poll
- **Reference Counting**: `kref` for safe context lifetime management

### Memory Barriers
- **Acquire Semantics**: `poll_wait()` provides acquire barrier
- **Proper Ordering**: Ensures counter reads occur after wait queue registration
- **Race Prevention**: Prevents missed wakeups in concurrent scenarios

### Recursion Prevention
- **Detection**: `current->in_eventfd` flag prevents recursive signals
- **Warning**: WARN_ON_ONCE alerts to potential deadlock conditions
- **Safe Contexts**: Callers should use `eventfd_signal_allowed()` to check safety

## Integration with Kernel Subsystems

### File System Integration
- **Anonymous Inodes**: Uses `anon_inodes` for file descriptor creation
- **Standard Operations**: Implements standard file operations (poll, read, write, release)
- **Process Integration**: Properly handles process termination cleanup

### I/O Multiplexing
- **poll/select**: Full support via `eventfd_poll()`
- **epoll**: Integrates seamlessly with epoll for scalable event handling
- **Edge-Triggered**: Supports both level and edge-triggered notifications

### Memory Management
- **Reference Counting**: Safe in multi-threaded environments
- **IDA Integration**: Efficient ID allocation and deallocation
- **Memory Barriers**: Proper ordering for lockless operations

## Error Handling

### System Call Errors
- **EAGAIN**: Returned when operation would block in non-blocking mode
- **EINVAL**: Invalid parameters or flags
- **EMFILE/ENFILE**: File descriptor table exhaustion

### Overflow Conditions
- **Graceful Degradation**: System continues operating at maximum counter value
- **Error Reporting**: EPOLLERR indicates overflow to applications
- **Recovery**: Counter can be reset by reading in normal mode

### Resource Exhaustion
- **ID Exhaustion**: IDA handles ID space exhaustion gracefully
- **Memory Allocation**: Standard kernel memory allocation error handling
- **File Descriptor Limits**: Respects per-process and system-wide limits

## Performance Characteristics

### Efficient Operations
- **Lockless Reads**: Poll operations avoid locks in common case
- **Minimal Memory**: Small context structure with efficient layout
- **Fast Wakeup**: Only wakes waiters when necessary

### Scalability
- **Per-Context Locking**: No global locks for independent eventfds
- **Reference Counting**: Scales well with multiple references
- **Wait Queue Efficiency**: Standard kernel wait queue infrastructure

This eventfd implementation provides a lightweight, efficient mechanism for event notification that integrates well with standard Unix I/O interfaces while providing the performance characteristics needed for high-throughput applications.