# io_uring.h - Core io_uring Header File

## Overview

The `io_uring.h` header file defines the core data structures, constants, and inline functions used throughout the io_uring subsystem. It serves as the central interface for io_uring operations and provides essential functionality for managing requests, completion queues, and various io_uring states.

## File Location
- **Path**: `io_uring/io_uring.h`
- **Purpose**: Core definitions and inline functions for io_uring
- **Dependencies**: Multiple kernel headers and io_uring specific modules

## Key Enumerations

### Return Code Constants
```c
enum {
    IOU_COMPLETE = 0,                    // Operation completed successfully
    IOU_ISSUE_SKIP_COMPLETE = -EIOCBQUEUED,  // Skip completion handling
    IOU_RETRY = -EAGAIN,                 // Retry operation (blocking or polling)
    IOU_REQUEUE = -3072,                 // Requeue task work for restart
};
```

### Status Flags
```c
enum {
    IO_CHECK_CQ_OVERFLOW_BIT,           // Check for CQ overflow
    IO_CHECK_CQ_DROPPED_BIT,            // Check for dropped events
};
```

## Core Data Structures

### io_wait_queue
```c
struct io_wait_queue {
    struct wait_queue_entry wq;         // Standard wait queue entry
    struct io_ring_ctx *ctx;            // Ring context
    unsigned cq_tail;                   // CQ tail when waiting started
    unsigned cq_min_tail;               // Minimum tail for wakeup
    unsigned nr_timeouts;               // Number of timeouts when waiting started
    int hit_timeout;                    // Whether timeout was hit
    ktime_t min_timeout;                // Minimum timeout value
    ktime_t timeout;                    // Current timeout value
    struct hrtimer t;                   // High-resolution timer
    
#ifdef CONFIG_NET_RX_BUSY_POLL
    ktime_t napi_busy_poll_dt;          // NAPI busy poll delta time
    bool napi_prefer_busy_poll;         // Prefer busy polling for NAPI
#endif
};
```

This structure manages waiting for completion events and handles timeouts.

## Important Constants

### Ring Limits
- **IORING_MAX_ENTRIES**: 32768 - Maximum submission queue entries
- **IORING_MAX_CQ_ENTRIES**: 65536 - Maximum completion queue entries (2x SQ)

## Core Functions

### Wait and Wake Functions

#### `io_should_wake(struct io_wait_queue *iowq)`
Determines if a waiting task should be woken up.
- **Parameters**: IO wait queue structure
- **Returns**: `true` if wake-up conditions are met
- **Logic**: 
  - Wakes if sufficient events are available (CQ tail advanced)
  - Wakes if timeout occurred since waiting began
  - Uses atomic read for timeout checking

#### `io_cqring_wake(struct io_ring_ctx *ctx)`
Wakes up all waiters on the completion queue wait queue.
- **Parameters**: Ring context
- **Wake Key**: `EPOLL_URING_WAKE | EPOLLIN`
- **Purpose**: Notifies userspace of new completion events
- **Dependency Detection**: Prevents infinite recursion in eventfd/epoll scenarios

#### `io_poll_wq_wake(struct io_ring_ctx *ctx)`
Wakes up poll waiters specifically.
- **Parameters**: Ring context
- **Wake Key**: `EPOLL_URING_WAKE | EPOLLIN`
- **Purpose**: Notify polling operations of readiness

### Ring State Functions

#### `io_sqring_full(struct io_ring_ctx *ctx)`
Checks if the submission queue is full.
- **Parameters**: Ring context
- **Returns**: `true` if SQ is full
- **Note**: Always reads actual ring head for SQPOLL safety

#### `io_sqring_entries(struct io_ring_ctx *ctx)`
Returns number of available submission queue entries.
- **Parameters**: Ring context
- **Returns**: Number of SQ entries ready for processing
- **Safety**: Uses `smp_load_acquire()` for proper ordering

### Completion Queue Management

#### `io_get_cqe_overflow(struct io_ring_ctx *ctx, struct io_uring_cqe **ret, bool overflow)`
Gets a completion queue entry, with overflow handling.
- **Parameters**: Context, CQE pointer, overflow flag
- **Returns**: `true` if CQE obtained successfully
- **Features**: 
  - Cache refill when needed
  - Support for 32-byte CQEs (CQE32)
  - Overflow tracking

#### `io_get_cqe(struct io_ring_ctx *ctx, struct io_uring_cqe **ret)`
Simplified CQE allocation without overflow handling.
- **Parameters**: Context and CQE pointer
- **Returns**: `true` on success
- **Usage**: Standard path for CQE allocation

#### `io_defer_get_uncommited_cqe(struct io_ring_ctx *ctx, struct io_uring_cqe **cqe_ret)`
Gets a CQE for deferred operations.
- **Parameters**: Context and CQE pointer
- **Returns**: `true` on success
- **Side Effect**: Sets `cq_flush` flag for batch processing

#### `io_fill_cqe_req(struct io_ring_ctx *ctx, struct io_kiocb *req)`
Fills a completion queue entry from a request.
- **Parameters**: Context and request
- **Returns**: `true` if CQE filled successfully
- **Features**:
  - Copies request CQE data to ring
  - Handles 32-byte CQE support
  - Triggers tracing if enabled
  - Clears big_cqe data after copy

### Request Management

#### `req_set_fail(struct io_kiocb *req)`
Marks a request as failed.
- **Parameters**: Request structure
- **Side Effects**:
  - Sets `REQ_F_FAIL` flag
  - Handles CQE skip logic
  - Sets link skip flags appropriately

#### `io_req_set_res(struct io_kiocb *req, s32 res, u32 cflags)`
Sets request result and completion flags.
- **Parameters**: Request, result code, completion flags
- **Purpose**: Store completion data in request structure

#### `io_req_complete_defer(struct io_kiocb *req)`
Defers request completion to batch processing.
- **Parameters**: Request structure
- **Requirement**: Must hold `uring_lock`
- **Purpose**: Add to deferred completion list for batching

#### `io_req_queue_tw_complete(struct io_kiocb *req, s32 res)`
Queues request for task work completion.
- **Parameters**: Request and result code
- **Purpose**: Complete request via task work mechanism

### Memory Management

#### `io_uring_alloc_async_data(struct io_alloc_cache *cache, struct io_kiocb *req)`
Allocates async data for a request.
- **Parameters**: Cache and request
- **Returns**: Allocated data pointer
- **Logic**: Uses cache if available, otherwise `kmalloc()`
- **Flag Setting**: Sets `REQ_F_ASYNC_DATA` on success

#### `req_has_async_data(struct io_kiocb *req)`
Checks if request has async data allocated.
- **Parameters**: Request structure
- **Returns**: `true` if async data present

### File Operations

#### `io_put_file(struct io_kiocb *req)`
Releases file reference from request.
- **Parameters**: Request structure
- **Logic**: Only calls `fput()` for non-fixed files

#### `io_file_can_poll(struct io_kiocb *req)`
Determines if request's file supports polling.
- **Parameters**: Request structure
- **Returns**: `true` if file supports polling
- **Caching**: Sets `REQ_F_CAN_POLL` flag for performance

### Lock Management

#### `io_ring_submit_lock(struct io_ring_ctx *ctx, unsigned issue_flags)`
Acquires submission lock if needed.
- **Parameters**: Context and issue flags
- **Logic**: Only locks for `IO_URING_F_UNLOCKED` operations
- **Usage**: Async worker threads need explicit locking

#### `io_ring_submit_unlock(struct io_ring_ctx *ctx, unsigned issue_flags)`
Releases submission lock if held.
- **Parameters**: Context and issue flags
- **Logic**: Only unlocks for `IO_URING_F_UNLOCKED` operations

### Task Work Functions

#### `io_run_task_work()`
Processes pending task work.
- **Returns**: `true` if work was processed
- **Features**:
  - Clears notification signals
  - Handles IO worker special case
  - Processes io_uring specific task work
  - Runs general task work

#### `io_task_work_pending(struct io_ring_ctx *ctx)`
Checks if task work is pending.
- **Parameters**: Context
- **Returns**: `true` if work pending
- **Checks**: Both general task work and local work lists

#### `io_local_work_pending(struct io_ring_ctx *ctx)`
Checks for local work pending.
- **Parameters**: Context
- **Returns**: `true` if local work available
- **Lists**: Checks both work and retry lists

### Request Allocation

#### `io_alloc_req(struct io_ring_ctx *ctx, struct io_kiocb **req)`
Allocates a request structure.
- **Parameters**: Context and request pointer
- **Returns**: `true` on successful allocation
- **Logic**: Uses cache, refills if empty

#### `io_extract_req(struct io_ring_ctx *ctx)`
Extracts request from free list cache.
- **Parameters**: Context
- **Returns**: Request structure
- **Usage**: Internal function for request recycling

#### `io_req_cache_empty(struct io_ring_ctx *ctx)`
Checks if request cache is empty.
- **Parameters**: Context
- **Returns**: `true` if cache empty

### Permission and Control Functions

#### `io_allowed_defer_tw_run(struct io_ring_ctx *ctx)`
Checks if deferred task work can run.
- **Parameters**: Context
- **Returns**: `true` if allowed
- **Logic**: Compares current task with submitter

#### `io_allowed_run_tw(struct io_ring_ctx *ctx)`
Checks if task work can run immediately.
- **Parameters**: Context
- **Returns**: `true` if allowed
- **Logic**: Considers defer flags and submitter task

#### `io_should_terminate_tw()`
Determines if task work should terminate.
- **Returns**: `true` if termination needed
- **Conditions**: `PF_KTHREAD` or `PF_EXITING` flags set

### Utility Functions

#### `io_commit_cqring(struct io_ring_ctx *ctx)`
Commits completion queue updates to ring.
- **Parameters**: Context
- **Implementation**: Uses `smp_store_release()` for ordering

#### `io_commit_cqring_flush(struct io_ring_ctx *ctx)`
Conditionally flushes completion queue updates.
- **Parameters**: Context
- **Conditions**: Timeout usage, eventfd, or poll activation

#### `io_get_task_refs(int nr)`
Manages task reference counting.
- **Parameters**: Number of references needed
- **Logic**: Uses cached refs, refills when needed

#### `uring_sqe_size(struct io_ring_ctx *ctx)`
Returns SQE size based on context flags.
- **Parameters**: Context
- **Returns**: SQE size (normal or 128-byte)

#### `io_get_time(struct io_ring_ctx *ctx)`
Gets current time based on context clock settings.
- **Parameters**: Context
- **Returns**: Current time
- **Clocks**: Supports monotonic and other clock types with offsets

#### `io_has_work(struct io_ring_ctx *ctx)`
Checks if context has pending work.
- **Parameters**: Context
- **Returns**: `true` if work available
- **Checks**: CQ overflow and local work pending

### Debugging and Lockdep

#### `io_lockdep_assert_cq_locked(struct io_ring_ctx *ctx)`
Asserts proper locking for completion queue operations.
- **Parameters**: Context
- **Conditions**: Different lock requirements based on setup flags:
  - `IORING_SETUP_DEFER_TASKRUN`: Requires `uring_lock`
  - `IORING_SETUP_IOPOLL`: Requires `uring_lock`
  - Default: Requires `completion_lock` or submitter task match

### Compatibility

#### `io_is_compat(struct io_ring_ctx *ctx)`
Checks if context is in compatibility mode.
- **Parameters**: Context
- **Returns**: `true` if in compat mode
- **Usage**: Handle 32-bit applications on 64-bit kernels

## Macros and Inline Utilities

### `io_for_each_link(pos, head)`
Iterates through linked requests.
- **Parameters**: Iterator variable and head request
- **Usage**: Process all requests in a link chain

### `io_tw_lock(struct io_ring_ctx *ctx, io_tw_token_t tw)`
Task work locking helper.
- **Parameters**: Context and task work token
- **Purpose**: Ensures proper locking in task work context

## External Declarations

### Global Variables
- `req_cachep`: External kmem_cache for request allocation

### Function Declarations
The header declares numerous functions implemented in other io_uring source files:
- Ring parameter functions
- CQE caching and posting
- Task work management
- File operations
- Request lifecycle management
- Polling operations
- Work queue integration

## Thread Safety and Ordering

### Memory Barriers
- Proper use of `smp_load_acquire()` and `smp_store_release()`
- Ordering guarantees for ring updates
- Safe concurrent access patterns

### Lock Dependencies
- Clear lock ordering requirements
- Context-specific locking strategies
- Lockdep integration for debugging

This header file is fundamental to io_uring's operation, providing the essential building blocks for high-performance asynchronous I/O operations in the Linux kernel.