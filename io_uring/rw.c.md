# rw.c - io_uring Read/Write Operations Implementation

## Overview

The `rw.c` file implements read and write operations for the io_uring subsystem. This includes handling both vectored and non-vectored I/O operations, buffer selection, async I/O management, and proper cleanup procedures. It serves as the core implementation for IORING_OP_READ, IORING_OP_WRITE, IORING_OP_READV, and IORING_OP_WRITEV operations.

## File Location
- **Path**: `io_uring/rw.c`
- **License**: GPL-2.0
- **Purpose**: Read/write operation handling for io_uring

## Key Data Structures

### io_rw
```c
struct io_rw {
    struct kiocb    kiocb;      // Kernel I/O control block (must be first)
    u64             addr;       // User buffer address
    u32             len;        // Buffer length
    rwf_t           flags;      // Read/write flags
};
```

This structure encapsulates a read or write operation, embedding the kernel's `kiocb` structure for integration with the VFS layer.

## Core Functions

### File Support Detection

#### `io_file_supports_nowait(struct io_kiocb *req, __poll_t mask)`
Determines if a file supports non-blocking I/O operations.
- **Parameters**: Request and poll mask
- **Returns**: `true` if file supports NOWAIT operations
- **Logic**:
  1. **FMODE_NOWAIT Check**: Returns true if `REQ_F_SUPPORT_NOWAIT` is set
  2. **Poll Check**: If file is pollable, checks current readiness via `vfs_poll()`
  3. **Fallback**: Returns false if neither NOWAIT nor polling is supported

### Buffer Selection and Preparation

#### `io_iov_buffer_select_prep(struct io_kiocb *req)`
Prepares buffer selection for vectored I/O operations.
- **Parameters**: I/O request
- **Returns**: 0 on success, negative error code on failure
- **Validation**: Ensures only single iovec for buffer selection
- **Compatibility**: Handles both native and compat mode iovecs

#### `io_iov_compat_buffer_select_prep(struct io_rw *rw)`
Handles buffer selection preparation for 32-bit compatibility mode.
- **Parameters**: Read/write structure
- **Returns**: 0 on success, -EFAULT on copy failure
- **Purpose**: Deals with 32-bit iovec structure on 64-bit kernels

### Buffer Import Functions

#### `io_import_vec(int ddir, struct io_kiocb *req, struct io_async_rw *io, const struct iovec __user *uvec, size_t uvec_segs)`
Imports user-space iovec arrays for vectored I/O operations.
- **Parameters**: 
  - `ddir`: Data direction (READ/WRITE)
  - `req`: I/O request
  - `io`: Async I/O structure
  - `uvec`: User iovec array
  - `uvec_segs`: Number of segments
- **Returns**: 0 on success, negative error on failure
- **Features**:
  - Uses existing iovec cache when available
  - Falls back to fast single iovec for simple cases
  - Handles compatibility mode differences
  - Marks request for cleanup when vectors allocated

#### `__io_import_rw_buffer(int ddir, struct io_kiocb *req, struct io_async_rw *io, unsigned int issue_flags)`
Core buffer import function for read/write operations.
- **Parameters**: Direction, request, async I/O structure, issue flags
- **Returns**: 0 on success, error code on failure
- **Logic**:
  1. **Vectored Check**: Calls `io_import_vec()` for vectored operations
  2. **Buffer Selection**: Handles dynamic buffer selection if enabled
  3. **Direct Import**: Uses `import_ubuf()` for simple buffer cases

#### `io_import_rw_buffer(int rw, struct io_kiocb *req, struct io_async_rw *io, unsigned int issue_flags)`
Wrapper function that imports buffers and saves iterator state.
- **Parameters**: Read/write direction, request, async I/O, flags
- **Returns**: 0 on success, error code on failure
- **Side Effect**: Saves iterator state for potential retries

### Resource Management

#### `io_rw_recycle(struct io_kiocb *req, unsigned int issue_flags)`
Recycles I/O resources for performance optimization.
- **Parameters**: Request and issue flags
- **Conditions**: Only recycles when not unlocked (single-threaded access)
- **Operations**:
  - KASAN poisoning for vector cache
  - Frees vectors exceeding soft capacity limit
  - Returns async data to allocation cache
  - Clears async data flags

#### `io_req_rw_cleanup(struct io_kiocb *req, unsigned int issue_flags)`
Performs cleanup for read/write requests.
- **Parameters**: Request and issue flags
- **Critical Issue**: Contains extensive comments about UAF (Use-After-Free) bug
- **Logic**:
  - Avoids recycling for reissued or reference-counted requests
  - Prevents cleanup parallelism with io-wq operations
  - Calls `io_rw_recycle()` when safe

#### `io_rw_alloc_async(struct io_kiocb *req)`
Allocates async data structure for read/write operations.
- **Parameters**: I/O request
- **Returns**: 0 on success, negative error on failure
- **Source**: Uses allocation cache from ring context for performance

### Completion Handlers

#### `io_complete_rw(struct kiocb *kiocb, long res)`
Standard completion handler for read/write operations.
- **Parameters**: Kernel I/O control block and result
- **Purpose**: Handles completion of async read/write operations
- **Integration**: Bridges VFS completion with io_uring completion system

#### `io_complete_rw_iopoll(struct kiocb *kiocb, long res)`
Completion handler specifically for IOPOLL operations.
- **Parameters**: Kernel I/O control block and result
- **Purpose**: Handles completion for polling-based I/O
- **Difference**: Optimized for polling scenarios

## Buffer Selection Architecture

### Dynamic Buffer Selection
The buffer selection mechanism allows applications to provide buffers at completion time rather than submission time, enabling more flexible memory management patterns.

#### Process Flow
1. **Submission**: Request submitted without specific buffer
2. **Selection**: Buffer selected from registered buffer group
3. **Operation**: I/O performed with selected buffer
4. **Completion**: Buffer ID returned with completion event

### Buffer Groups
- Buffers organized into groups for efficient selection
- Applications register buffer groups with specific IDs
- Kernel selects appropriate buffer based on operation requirements

## Vectored I/O Support

### iovec Handling
- Support for both single and multiple iovec structures
- Efficient caching of iovec arrays for repeated operations
- Proper cleanup and recycling of vector resources

### Compatibility Layer
- Native 64-bit iovec handling
- 32-bit compatibility mode support
- Transparent handling of different userspace architectures

## Error Handling and Edge Cases

### Use-After-Free Prevention
The code contains extensive comments and logic to prevent UAF conditions that can occur when:
- I/O operations are offloaded to io-wq worker threads
- Cleanup runs in parallel with ongoing operations
- Core VFS code accesses iovec data after completion

### Race Condition Mitigation
- Careful ordering of cleanup operations
- Reference counting for shared resources
- Proper synchronization with worker thread operations

## Performance Optimizations

### Resource Recycling
- Aggressive recycling of async data structures
- Vector cache management with soft capacity limits
- KASAN integration for debugging without performance loss

### Fast Path Optimization
- Single iovec fast path for common cases
- Cache-friendly data structure layouts
- Minimal allocation in hot paths

### NOWAIT Support
- Efficient detection of non-blocking capability
- Fallback to polling when NOWAIT unavailable
- Integration with file system NOWAIT support

## Integration Points

### VFS Layer Integration
- Uses standard `kiocb` structure for VFS compatibility
- Proper handling of VFS return codes and semantics
- Integration with file system specific optimizations

### io_uring Core Integration
- Seamless integration with completion queue system
- Proper request lifecycle management
- Support for all io_uring features (linking, cancellation, etc.)

### Memory Management Integration
- Integration with io_uring allocation caches
- Proper handling of user memory mapping
- NUMA-aware memory operations where applicable

## Debugging and Observability

### KASAN Integration
- Memory poisoning for debugging use-after-free issues
- Comprehensive coverage of recycled structures
- No performance impact in production builds

### Error Path Coverage
- Comprehensive error handling for all allocation failures
- Proper cleanup on partial operation completion
- Clear error propagation to userspace

## Thread Safety

### Locking Strategy
- Minimal locking through careful data structure design
- Per-request data isolation
- Safe sharing of read-only resources

### Async Context Handling
- Safe operation in both sync and async contexts
- Proper handling of context switches during I/O
- Worker thread safety considerations

This implementation provides the foundation for high-performance read and write operations in io_uring, balancing performance, safety, and flexibility while maintaining compatibility with the existing VFS infrastructure.