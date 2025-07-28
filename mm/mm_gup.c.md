# Linux Kernel Get User Pages (`mm/gup.c`)

## Overview

The `mm/gup.c` file implements Linux's Get User Pages (GUP) subsystem, a critical memory management component that provides fast and safe access to user-space memory from kernel context. This system enables kernel code, device drivers, and I/O subsystems to efficiently pin user pages in memory, preventing them from being swapped out or migrated during critical operations like DMA transfers, network I/O, and inter-process communication.

## Core Architecture

### 1. Page Following Infrastructure

**Follow Page Context** - Lines 31-34:
```c
struct follow_page_context {
    struct dev_pagemap *pgmap;    // Device memory page mapping cache
    unsigned int page_mask;       // Output page size mask
};
```

**Purpose**: Efficiently traverses page tables to locate and validate user pages across different memory types including regular pages, huge pages, and device memory.

### 2. Reference Counting Models

**FOLL_GET vs FOLL_PIN**:
- **`FOLL_GET`**: Traditional reference counting for temporary access
- **`FOLL_PIN`**: Enhanced pinning for long-term usage (DMA, file I/O)
- **Pin Counting**: Uses separate pin counter for better tracking of long-term references

**`try_grab_folio()`** - Lines 145-179:
- **Reference Management**: Handles both GET and PIN reference models
- **Zero Page Optimization**: Avoids pinning the ubiquitous zero page
- **PCI P2PDMA Support**: Handles peer-to-peer device memory access
- **Pin Statistics**: Tracks pinned page statistics for monitoring

## Fast GUP vs Slow GUP

### 1. Page Table Walking

**Multi-level Page Table Traversal**:
- **`follow_page_mask()`** - Lines 1067-1088: Main page following entry point
- **`follow_p4d_mask()`** - Lines 1026-1041: P4D level traversal
- **`follow_pud_mask()`** - Lines 999-1024: PUD level with huge page support
- **`follow_pmd_mask()`**: PMD level with transparent huge page handling

**Optimization Strategy**:
```
Fast Path: Lockless page table walk with atomic operations
Slow Path: Full fault handling with mmap_lock held
```

### 2. Fault Integration

**`faultin_page()`** - Lines 1147-1199:
- **Fault Flag Translation**: Converts GUP flags to fault flags
- **Retry Logic**: Handles VM_FAULT_RETRY and lock management
- **Unshare Support**: Implements copy-on-write breaking for pinning
- **Signal Handling**: Supports interruptible and killable operations

## Pin vs Get Semantics

### 1. Pin Lifecycle Management

**`unpin_user_page()`** - Lines 190-195:
- **Pin Validation**: Sanity checks for pinned anonymous pages
- **Reference Cleanup**: Properly decrements pin counts and references
- **Statistics Update**: Updates NR_FOLL_PIN_RELEASED counters

**`gup_put_folio()`** - Lines 107-120:
- **Pin Count Handling**: Manages separate pin counter for large folios
- **Bias Accounting**: Uses GUP_PIN_COUNTING_BIAS for reference scaling
- **Zero Page Protection**: Avoids operations on the shared zero page

### 2. Folio Pin Management

**`try_get_folio()`** - Lines 79-105:
- **Stability Guarantee**: Ensures folio stability during reference acquisition
- **Split Detection**: Handles folio splitting during reference operations
- **Retry Logic**: Robust handling of concurrent folio operations

## User Interface Functions

### 1. Core GUP Functions

**`get_user_pages_remote()`** - Lines 2639-2654:
- **Remote Process Access**: Access pages from different mm_struct
- **Flexible Locking**: Supports unlockable operations
- **Security Integration**: Respects process boundaries and permissions
- **Legacy Support**: Maintains compatibility with existing callers

**`get_user_pages()`** - Lines 2680-2691:
- **Current Process**: Simplified interface for current->mm
- **Automatic Locking**: Handles mmap_lock internally
- **Common Use Case**: Most frequent GUP usage pattern

### 2. Long-term Pinning

**`__gup_longterm_locked()`** - Lines 2501-2530:
- **Movable Page Migration**: Ensures pinned pages don't block memory management
- **CMA Integration**: Handles Contiguous Memory Allocator constraints
- **Retry Logic**: Migrates movable pages and retries pinning
- **Memory Allocation Context**: Uses special memory allocation flags

## Memory Types and Special Cases

### 1. Device Memory Support

**Zone Device Integration**:
- **Device Pagemap Caching**: Avoids repeated device memory lookups
- **P2P DMA Support**: Handles peer-to-peer device memory transfers
- **Migration Restrictions**: Prevents migration of device memory

### 2. Huge Page Handling

**Transparent Huge Page Support**:
- **PMD-level Operations**: Direct handling of 2MB huge pages
- **PUD-level Support**: Support for 1GB huge pages where available
- **Split Coordination**: Handles THP splitting during GUP operations

### 3. Special Memory Regions

**Gate Page Handling** - Lines 1090-1140:
- **VDSO Access**: Handles virtual dynamic shared object pages
- **Read-only Enforcement**: Prevents writes to gate pages
- **Architecture Integration**: Works with architecture-specific gate implementations

## Security and Validation

### 1. Pin Validation

**`sanity_check_pinned_pages()`** - Lines 36-73:
- **Anonymous Exclusivity**: Validates exclusive anonymous page ownership
- **THP Consistency**: Ensures proper THP pinning semantics
- **Debug Support**: Provides comprehensive debugging for pin operations

### 2. Argument Validation

**`is_valid_gup_args()`** - Lines 2536-2580:
- **Flag Validation**: Ensures only valid flag combinations
- **Internal Flag Protection**: Prevents external use of internal flags
- **Mutual Exclusion**: Enforces FOLL_GET/FOLL_PIN exclusivity
- **Parameter Consistency**: Validates argument relationships

## Performance Optimizations

### 1. Lockless Operations

**Fast GUP Path**:
- **Atomic Reference Updates**: Lock-free page reference management
- **RCU Protection**: Safe page table traversal without locks
- **Speculation Support**: Handles speculative page table walks

### 2. Batch Processing

**Folio Batch Operations**:
- **Reduced Lock Contention**: Batches operations to minimize locking
- **Cache Efficiency**: Improves CPU cache utilization
- **Scalability**: Better performance on multi-core systems

## Integration Points

### 1. VM Subsystem Integration

**Memory Management Coordination**:
- **Page Reclaim**: Prevents reclaim of pinned pages
- **Migration Blocking**: Coordinates with page migration
- **Compaction**: Handles memory compaction constraints

### 2. I/O Subsystem Support

**DMA Integration**:
- **Zero-copy I/O**: Enables direct user memory access
- **Device Driver Support**: Provides stable memory for hardware access
- **Network Stack**: Supports sendfile and splice operations

### 3. File System Integration

**Direct I/O Support**:
- **Page Cache Bypass**: Enables direct user memory I/O
- **Write-through Guarantees**: Ensures data consistency
- **mmap Integration**: Coordinates with memory-mapped I/O

## Error Handling and Recovery

### 1. Fault Handling

**Robust Error Recovery**:
- **Partial Success**: Handles partial page acquisition
- **Signal Interruption**: Supports interruptible operations
- **Lock Recovery**: Proper mmap_lock handling on errors

### 2. Resource Cleanup

**Reference Management**:
- **Automatic Cleanup**: Ensures proper reference counting
- **Exception Paths**: Handles allocation failures gracefully
- **Memory Pressure**: Adapts behavior under memory pressure

## Debugging and Observability

### 1. Statistics Tracking

**Pin Statistics**:
- **NR_FOLL_PIN_ACQUIRED**: Tracks successful pin operations
- **NR_FOLL_PIN_RELEASED**: Monitors pin release operations
- **Per-node Accounting**: Provides NUMA-aware statistics

### 2. Debug Infrastructure

**Validation Framework**:
- **VM_BUG_ON Checks**: Comprehensive invariant validation
- **Pin State Tracking**: Monitors pin state consistency
- **Reference Debugging**: Tracks reference count operations

## Advanced Features

### 1. NUMA Awareness

**Node-local Operations**:
- **Preferred Node Access**: Optimizes for local memory access
- **Migration Coordination**: Handles NUMA balancer integration
- **Cross-node References**: Manages remote node page access

### 2. Memory Cgroup Integration

**Resource Accounting**:
- **Pin Accounting**: Tracks pinned memory per cgroup
- **Limit Enforcement**: Respects memory cgroup limits
- **Hierarchical Control**: Supports cgroup hierarchies

### 3. Real-time Support

**Deterministic Behavior**:
- **Priority Inheritance**: Supports RT task requirements
- **Lock Ordering**: Maintains consistent lock ordering
- **Preemption Points**: Provides safe preemption points

## Configuration and Tuning

### 1. Compile-time Options

**Feature Selection**:
- **CONFIG_MMU**: Memory management unit support
- **CONFIG_ZONE_DEVICE**: Device memory support
- **CONFIG_TRANSPARENT_HUGEPAGE**: THP integration

### 2. Runtime Parameters

**Behavioral Control**:
- **Pin limits**: Controls maximum pinned memory
- **Migration policies**: Configures page migration behavior
- **Performance tuning**: Adjusts for specific workloads

The GUP implementation represents a sophisticated balance between performance, safety, and flexibility, providing the foundation for high-performance I/O operations while maintaining memory management correctness and security boundaries. Its support for diverse memory types and usage patterns makes it an essential component of modern Linux memory management.