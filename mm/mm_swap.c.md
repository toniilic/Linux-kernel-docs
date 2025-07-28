# Linux Kernel Page Management and LRU Operations (`mm/swap.c`)

## Overview

The `mm/swap.c` file implements core Linux kernel page management operations, focusing on Least Recently Used (LRU) list management, page reference handling, and folio lifecycle operations. This subsystem forms the foundation of memory reclaim policies, providing efficient mechanisms for tracking page usage patterns and coordinating between different memory management components. Despite its name, this file handles general page operations, not just swap-related functionality.

## Core Architecture

### 1. Per-CPU Folio Batching

**CPU-Local Batch Structure** - Lines 50-71:
```c
struct cpu_fbatches {
    local_lock_t lock;                    // Protects preempt-disabled batches
    struct folio_batch lru_add;           // New folios to add to LRU
    struct folio_batch lru_deactivate_file; // File folios to deactivate
    struct folio_batch lru_deactivate;    // General deactivation batch
    struct folio_batch lru_lazyfree;      // Lazy-free folios
    struct folio_batch lru_activate;      // Folios to activate (SMP only)
    
    local_lock_t lock_irq;                // IRQ-disabled protection
    struct folio_batch lru_move_tail;     // Move folios to tail
};
```

**Purpose**: Reduces lock contention by batching LRU operations per CPU, improving scalability and performance in high-concurrency scenarios.

### 2. Page Clustering Configuration

**Swap Readahead Control** - Lines 46-48:
```c
int page_cluster;                    // Power of 2 pages to swap together
static const int page_cluster_max = 31;  // Maximum cluster size
```

**Optimization**: Controls swap I/O efficiency by grouping related pages for batch processing.

## Folio Lifecycle Management

### 1. Folio Reference Counting

**`__folio_put()`** - Lines 97-114:
- **Zone Device Handling**: Special processing for device memory
- **Hugetlb Integration**: Dedicated huge page freeing path
- **LRU Cleanup**: Removes folios from LRU before freeing
- **Memory Cgroup Integration**: Uncharges memory from cgroups
- **Deferred Split Handling**: Manages transparent huge page splitting

**Reference Management Chain**:
1. **Reference Drop**: Atomic decrement of folio reference count
2. **LRU Removal**: Remove from appropriate LRU list
3. **Cgroup Uncharge**: Update memory cgroup accounting
4. **Memory Return**: Return pages to buddy allocator

### 2. Batch Operations

**`folios_put_refs()`** - Lines 941-994:
- **Optimized Batching**: Efficiently processes multiple folios
- **Lock Optimization**: Minimizes lock acquisitions through batching
- **Special Case Handling**: Zone devices, huge pages, huge zero page
- **Memory Cgroup Batch Uncharge**: Bulk cgroup memory operations

## LRU List Management

### 1. LRU Addition

**`lru_add()`** - Lines 118-156:
- **Evictability Check**: Determines if folio can be reclaimed
- **Unevictable Handling**: Manages mlocked and non-reclaimable pages
- **Statistics Updates**: Updates VM events for monitoring
- **Mlock Integration**: Coordinates with memory locking subsystem

**Folio State Transitions**:
- **New → Inactive**: Default path for new folios
- **Unevictable → Rescued**: When folio becomes evictable
- **Evictable → Unevictable**: When folio becomes locked/pinned

### 2. LRU Manipulation Functions

**`folio_mark_accessed()`** - Lines 449-483:
- **Reference Tracking**: Implements two-level reference tracking
- **LRU Generation Support**: Integrates with multi-generational LRU
- **Activation Logic**: Promotes folios through LRU hierarchy
- **Working Set Tracking**: Coordinates with working set size detection

**State Machine**:
```
inactive,unreferenced → inactive,referenced → active,unreferenced → active,referenced
```

### 3. Deactivation Operations

**`lru_deactivate_file()`** - Lines 548-587:
- **File-specific Logic**: Handles file-backed page deactivation
- **Writeback Optimization**: Moves dirty pages to reclaim position
- **Mapping Check**: Avoids deactivating actively mapped pages
- **Performance Counters**: Updates deactivation statistics

**`lru_deactivate()`** - Lines 589-602:
- **General Deactivation**: Handles both file and anonymous pages
- **Active → Inactive**: Demotes overused active pages
- **Reference Clearing**: Resets reference tracking state

## Multi-Generational LRU Integration

### 1. LRU Generation Support

**`lru_gen_inc_refs()`** - Lines 380-405:
- **Reference Increment**: Atomic reference count management
- **Generation Tracking**: Maintains generational LRU metadata
- **Overflow Protection**: Handles reference count saturation
- **Lock-free Updates**: Uses atomic operations for scalability

**`lru_gen_clear_refs()`** - Lines 407-421:
- **Reference Reset**: Clears accumulated reference information
- **Working Set Integration**: Coordinates with working set detection
- **Generation Check**: Validates generation consistency

### 2. Conditional Compilation

**Feature Toggles** - Lines 423-434:
```c
#ifdef CONFIG_LRU_GEN
    // Multi-generational LRU implementation
#else
    // Stub implementations for disabled feature
#endif
```

## Batch Processing Infrastructure

### 1. Movement Functions

**`folio_batch_move_lru()`** - Lines 158-176:
- **Batch Processing**: Efficiently processes folio batches
- **Lock Coalescing**: Minimizes LRU lock acquisitions
- **Reference Management**: Handles folio reference counting
- **Trace Integration**: Provides tracing for debugging

**`__folio_batch_add_and_move()`** - Lines 178-201:
- **Conditional Batching**: Handles large folios immediately
- **LRU Cache State**: Respects LRU cache disable state
- **Interrupt Handling**: Provides IRQ-safe and normal variants

### 2. VMA-Aware Operations

**`folio_add_lru_vma()`** - Lines 517-525:
- **VMA Context**: Considers virtual memory area properties
- **Mlock Integration**: Handles VM_LOCKED memory regions
- **Special Memory**: Avoids LRU for special mappings

## System-Wide LRU Operations

### 1. LRU Draining

**`lru_add_drain_all()`** - Lines 802-884:
- **Cross-CPU Coordination**: Synchronizes LRU operations across CPUs
- **Generation Tracking**: Prevents redundant drain operations
- **Work Queue Integration**: Uses kernel work queues for coordination
- **Memory Barriers**: Ensures proper ordering of operations

**Sophisticated Algorithm**:
1. **Load Current Generation**: Check if drain already in progress
2. **Increment Generation**: Mark new drain operation
3. **CPU Scan**: Identify CPUs needing drain operations
4. **Work Dispatch**: Queue drain work on relevant CPUs
5. **Synchronization**: Wait for all drain operations to complete

### 2. LRU Cache Control

**`lru_cache_disable()`** - Lines 902-924:
- **Migration Preparation**: Disables LRU for page migration
- **RCU Synchronization**: Ensures all CPUs see disable state
- **Forced Draining**: Drains all CPU caches before disable
- **Reference Counting**: Uses atomic counter for nesting

**Critical for**:
- **Page Migration**: Ensures pages don't move during migration
- **Memory Hotplug**: Coordinates with memory addition/removal
- **Compaction**: Supports memory defragmentation operations

## Performance Optimizations

### 1. Lock Granularity

**Local Locks** - Lines 55, 64:
```c
local_lock_t lock;        // Preemption protection
local_lock_t lock_irq;    // IRQ protection
```

**Benefits**:
- **Reduced Contention**: Per-CPU locking reduces contention
- **RT Compatibility**: Works with real-time kernels
- **Fine-grained Control**: Separate locks for different contexts

### 2. Batch Size Optimization

**Adaptive Batching**:
- **Large Folio Handling**: Processes large folios immediately
- **Cache Line Efficiency**: Optimizes for CPU cache performance
- **Memory Pressure Response**: Adjusts batch sizes under pressure

## Integration Points

### 1. Memory Cgroups

**Cgroup Integration**:
- **Accounting**: Tracks memory usage per cgroup
- **Limit Enforcement**: Respects cgroup memory limits
- **Statistics**: Provides per-cgroup LRU statistics
- **Hierarchical Support**: Handles cgroup hierarchies

### 2. NUMA Awareness

**NUMA Integration**:
- **Node-local Operations**: Prefers local memory operations
- **Cross-node Coordination**: Handles multi-node systems
- **Migration Support**: Coordinates with NUMA balancing

### 3. Transparent Huge Pages

**THP Integration**:
- **Split Coordination**: Handles THP splitting operations
- **Compound Page Support**: Manages multi-page folios
- **Deferred Operations**: Queues operations for later processing

## Error Handling and Robustness

### 1. State Validation

**Consistency Checks**:
- **VM_BUG_ON**: Validates critical invariants
- **Reference Counting**: Ensures proper reference management
- **LRU State**: Validates LRU membership consistency

### 2. Graceful Degradation

**Fallback Mechanisms**:
- **Batch Overflow**: Handles batch size limitations
- **Memory Pressure**: Adapts to low memory conditions
- **CPU Hotplug**: Handles CPU addition/removal

## Debugging and Observability

### 1. Tracing Support

**Trace Points**:
- **LRU Operations**: Traces folio movement between lists
- **Reference Updates**: Tracks reference count changes
- **Batch Processing**: Monitors batch operation efficiency

### 2. Statistics

**VM Events**:
- **PGDEACTIVATE**: Tracks page deactivation events
- **PGROTATED**: Monitors page rotation in LRU
- **UNEVICTABLE_PGRESCUED**: Tracks unevictable page rescues

## Security Considerations

### 1. Reference Count Integrity

**Protection Mechanisms**:
- **Atomic Operations**: Prevents race conditions
- **Overflow Detection**: Handles reference count overflow
- **Use-after-free Prevention**: Ensures safe memory access

### 2. Access Control

**Memory Protection**:
- **Privilege Validation**: Respects memory access permissions
- **Isolation**: Maintains process memory isolation
- **Cgroup Boundaries**: Enforces memory cgroup limits

The swap.c implementation represents a sophisticated balance between performance and correctness, providing the fundamental page management operations that enable efficient memory reclaim and ensure system stability under various memory pressure scenarios. Its integration with modern features like multi-generational LRU and transparent huge pages makes it a critical component of Linux's advanced memory management capabilities.