# mm/memory.c - Linux Core Virtual Memory Management

## Overview

This file implements the core virtual memory management system for Linux, originally developed by Linus Torvalds in 1991-1994. It provides the fundamental infrastructure for virtual memory operations including page fault handling, memory mapping, copy-on-write semantics, and virtual-to-physical address translation. The implementation supports demand paging, shared memory, swap operations, and modern features like transparent huge pages, NUMA balancing, and memory protection keys.

## Historical Development

### Key Milestones and Contributors
- **Linus Torvalds (1991-1994)**: Original virtual memory implementation
  - **December 1, 1991**: Initial demand-loading implementation
  - **December 2, 1991**: Shared pages support
  - **December 18-20, 1991**: Real VM with paging to/from disk
- **Alex Bligh (1994)**: Multi-page memory management for v1.1
- **Gerhard Wichert - Siemens AG (1999)**: BIGMEM support
- **Andi Kleen (2004)**: Four-level page tables

### Evolution Timeline
- **1991**: Basic demand paging and shared memory
- **1994**: Multi-page management
- **1999**: High memory (HIGHMEM) support
- **2004**: Four-level page table hierarchy
- **2010s**: Transparent huge pages, NUMA balancing
- **2020s**: Memory protection keys, user fault handling

### Design Philosophy
The memory management system is built around demand paging principles, copy-on-write optimization, and hierarchical page table management, providing efficient virtual memory with minimal physical memory usage while maintaining security and performance.

## Core Concepts

### Virtual Memory Architecture

#### Page Fault Handling Pipeline
```
Hardware Exception → Page Fault Handler → VMA Lookup → Fault Resolution
       ↓                    ↓                ↓             ↓
[CPU MMU Exception]  [handle_mm_fault()]  [VMA Validation] [Page Allocation/COW]
```

#### Memory Management Hierarchy
```
Process Virtual Memory
  ↓
Virtual Memory Areas (VMAs)
  ↓
Page Tables (PGD→P4D→PUD→PMD→PTE)
  ↓
Physical Pages
  ↓
Hardware Memory Management Unit (MMU)
```

#### Virtual Memory Types
- **Anonymous Memory**: Process heap, stack, and dynamically allocated memory
- **File-Backed Memory**: Memory-mapped files and executable code
- **Shared Memory**: IPC shared memory segments and shared libraries
- **Device Memory**: Memory-mapped I/O and special device mappings

## Key Data Structures

### `struct vm_fault` - Page Fault Context
```c
struct vm_fault {
    struct vm_area_struct *vma;        /* Target VMA */
    unsigned long address;             /* Faulting address (page-aligned) */
    unsigned long real_address;        /* Original faulting address */
    unsigned int flags;                /* Fault flags */
    
    pgoff_t pgoff;                     /* Logical page offset */
    gfp_t gfp_mask;                   /* Allocation flags */
    
    pgd_t *pgd;                       /* Page global directory */
    p4d_t *p4d;                       /* Page 4th level directory */
    pud_t *pud;                       /* Page upper directory */
    pmd_t *pmd;                       /* Page middle directory */
    pte_t *pte;                       /* Page table entry */
    
    pte_t orig_pte;                   /* Original PTE value */
    pmd_t orig_pmd;                   /* Original PMD value */
    
    spinlock_t *ptl;                  /* Page table lock */
    struct page *cow_page;            /* COW target page */
    struct page *page;                /* Fault target page */
};
```

### Page Fault Flags
```c
/* Fault type flags */
#define FAULT_FLAG_WRITE        0x01    /* Write fault */
#define FAULT_FLAG_MKWRITE      0x02    /* Make page writable */
#define FAULT_FLAG_ALLOW_RETRY  0x04    /* Allow retry */
#define FAULT_FLAG_RETRY_NOWAIT 0x08    /* Non-blocking retry */
#define FAULT_FLAG_KILLABLE     0x10    /* Killable fault */
#define FAULT_FLAG_TRIED        0x20    /* Second try */
#define FAULT_FLAG_USER         0x40    /* User-space fault */
#define FAULT_FLAG_REMOTE       0x80    /* Remote fault */
#define FAULT_FLAG_INSTRUCTION  0x100   /* Instruction fetch */
#define FAULT_FLAG_INTERRUPTIBLE 0x200  /* Interruptible fault */
#define FAULT_FLAG_UNSHARE      0x400   /* Unshare COW page */
#define FAULT_FLAG_ORIG_PTE_VALID 0x800 /* Original PTE valid */
#define FAULT_FLAG_VMA_LOCK     0x1000  /* VMA lock held */
```

### Virtual Memory Fault Results
```c
typedef unsigned int vm_fault_t;

/* Fault result flags */
#define VM_FAULT_OOM        0x0001    /* Out of memory */
#define VM_FAULT_SIGBUS     0x0002    /* Bad address */
#define VM_FAULT_MAJOR      0x0004    /* Major fault */
#define VM_FAULT_HWPOISON   0x0010    /* Hardware poisoned page */
#define VM_FAULT_HWPOISON_LARGE 0x0020 /* Poisoned large page */
#define VM_FAULT_SIGSEGV    0x0040    /* Segmentation violation */
#define VM_FAULT_NOPAGE     0x0100    /* No page mapping */
#define VM_FAULT_LOCKED     0x0200    /* Page locked */
#define VM_FAULT_RETRY      0x0400    /* Retry fault */
#define VM_FAULT_FALLBACK   0x0800    /* Fallback to smaller page */
#define VM_FAULT_DONE_COW   0x1000    /* COW completed */
#define VM_FAULT_NEEDDSYNC  0x2000    /* Datasync required */
#define VM_FAULT_COMPLETED  0x4000    /* Fault completed */
#define VM_FAULT_HINDEX_MASK 0xf0000  /* Huge page index mask */
```

## Core Functions

### Page Fault Processing

#### `handle_mm_fault()` - Main Page Fault Entry Point
```c
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, 
                          unsigned long address, 
                          unsigned int flags)
```

**Purpose**: Primary entry point for handling memory management faults from architecture-specific code

**Fault Processing Steps**:
1. **Context Setup**: Initialize fault context and validate parameters
2. **VMA Validation**: Verify VMA permissions and accessibility
3. **Fault Accounting**: Track fault statistics and performance metrics
4. **Fault Delegation**: Delegate to internal fault handling mechanism
5. **Result Processing**: Process fault results and handle retries

**Key Features**:
- **Multi-threading Safety**: Proper locking and synchronization
- **Performance Optimization**: Fast-path optimization for common cases
- **Error Handling**: Comprehensive error handling and recovery
- **Statistics Integration**: Integration with system performance monitoring

#### `__handle_mm_fault()` - Internal Fault Handler
```c
static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
                                   unsigned long address, 
                                   unsigned int flags)
```

**Page Table Hierarchy Navigation**:
1. **PGD Level**: Page Global Directory lookup and allocation
2. **P4D Level**: Page 4th Level Directory (5-level paging)
3. **PUD Level**: Page Upper Directory with huge page support
4. **PMD Level**: Page Middle Directory with transparent huge pages
5. **PTE Level**: Page Table Entry handling

**Huge Page Handling**:
```c
/* PUD-level huge pages */
if (pud_none(*vmf.pud) && thp_vma_allowable_order(vma, vm_flags,
                    TVA_IN_PF | TVA_ENFORCE_SYSFS, PUD_ORDER)) {
    ret = create_huge_pud(&vmf);
    if (!(ret & VM_FAULT_FALLBACK))
        return ret;
}

/* PMD-level huge pages */
if (pmd_none(*vmf.pmd) && thp_vma_allowable_order(vma, vm_flags,
                    TVA_IN_PF | TVA_ENFORCE_SYSFS, PMD_ORDER)) {
    ret = create_huge_pmd(&vmf);
    if (!(ret & VM_FAULT_FALLBACK))
        return ret;
}
```

#### `handle_pte_fault()` - Page Table Entry Fault Handler
```c
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
```

**PTE Fault Classification**:
1. **Missing PTE**: No page table entry exists
2. **Non-Present PTE**: Page swapped out or not yet allocated
3. **Protection Fault**: Permission violation (write to read-only)
4. **NUMA Fault**: NUMA balancing page migration
5. **Access Fault**: First access to page (young bit setting)

**Fault Routing Logic**:
```c
if (!vmf->pte)
    return do_pte_missing(vmf);        /* Missing PTE */

if (!pte_present(vmf->orig_pte))
    return do_swap_page(vmf);          /* Swapped page */

if (pte_protnone(vmf->orig_pte) && vma_is_accessible(vmf->vma))
    return do_numa_page(vmf);          /* NUMA balancing */

if (vmf->flags & (FAULT_FLAG_WRITE|FAULT_FLAG_UNSHARE)) {
    if (!pte_write(entry))
        return do_wp_page(vmf);        /* Copy-on-write */
}
```

### Anonymous Memory Management

#### `do_anonymous_page()` - Anonymous Page Fault Handler
```c
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
```

**Anonymous Page Allocation Process**:
1. **Zero Page Optimization**: Use shared zero page for read-only faults
2. **Private Page Allocation**: Allocate private anonymous page for writes
3. **Page Table Setup**: Install page in page table with appropriate permissions
4. **Memory Accounting**: Update RSS and memory statistics
5. **NUMA Placement**: Consider NUMA topology for page placement

**Zero Page Optimization**:
```c
/* Use the zero-page for reads */
if (!(vmf->flags & FAULT_FLAG_WRITE) &&
        !mm_forbids_zeropage(vma->vm_mm)) {
    entry = pte_mkspecial(pfn_pte(my_zero_pfn(vmf->address),
                                 vma->vm_page_prot));
    /* Install zero page PTE */
    goto setpte;
}
```

**Private Page Allocation**:
```c
/* Allocate our own private page */
ret = vmf_anon_prepare(vmf);
if (ret)
    return ret;

folio = alloc_anon_folio(vmf);
if (IS_ERR(folio))
    return PTR_ERR(folio);
```

#### Anonymous Memory Features
- **Lazy Allocation**: Pages allocated only when accessed
- **Zero Page Sharing**: Shared zero page for unmodified anonymous memory
- **NUMA Awareness**: NUMA-aware page allocation
- **Memory Accounting**: Accurate RSS accounting
- **Swap Support**: Integration with swap subsystem

### Copy-on-Write (COW) Implementation

#### `do_wp_page()` - Write Protection Fault Handler
**Purpose**: Handle copy-on-write faults when writing to shared or protected pages

**COW Scenarios**:
1. **Fork COW**: Parent and child sharing pages after fork()
2. **mmap COW**: MAP_PRIVATE file mappings
3. **Write Protection**: Software write protection
4. **NUMA COW**: NUMA balancing write faults

**COW Process**:
1. **Page Analysis**: Determine if COW is required
2. **Page Allocation**: Allocate new page for private copy
3. **Page Copy**: Copy data from shared to private page
4. **PTE Update**: Update page table entry to point to new page
5. **TLB Invalidation**: Invalidate TLB entries for consistency

#### COW Optimizations
- **Page Reuse**: Reuse page if no other references exist
- **Large Page COW**: COW handling for transparent huge pages
- **NUMA Optimization**: NUMA-aware COW page placement
- **Memory Deduplication**: Integration with KSM (Kernel Same-page Merging)

### Swap and Page Reclaim Integration

#### `do_swap_page()` - Swap Page Fault Handler
**Purpose**: Handle faults on pages that have been swapped out to storage

**Swap-in Process**:
1. **Swap Entry Validation**: Validate swap entry and swap device
2. **Page Allocation**: Allocate physical page for swap-in
3. **Swap Read**: Read page data from swap device
4. **Page Installation**: Install page in page table
5. **Swap Space Cleanup**: Free swap space entry

**Swap Optimizations**:
- **Swap Cache**: Cache swapped pages for performance
- **Readahead**: Predictive swap readahead
- **NUMA Placement**: NUMA-aware swap-in page placement
- **Compression**: Integration with swap compression (zswap)

### NUMA Balancing

#### `do_numa_page()` - NUMA Balancing Fault Handler
**Purpose**: Handle NUMA balancing faults for automatic NUMA optimization

**NUMA Migration Process**:
1. **Access Tracking**: Track page access patterns
2. **Migration Decision**: Decide whether to migrate page
3. **Target Selection**: Select optimal NUMA node
4. **Page Migration**: Migrate page to target node
5. **Statistics Update**: Update NUMA balancing statistics

**NUMA Features**:
- **Automatic Balancing**: Automatic NUMA optimization
- **Access Tracking**: Track memory access patterns
- **Migration Heuristics**: Intelligent migration decisions
- **Performance Monitoring**: Detailed NUMA statistics

### Memory Protection and Security

#### User Fault Handling Integration
```c
/* Userfaultfd integration */
if (userfaultfd_missing(vma)) {
    pte_unmap_unlock(vmf->pte, vmf->ptl);
    return handle_userfault(vmf, VM_UFFD_MISSING);
}
```

**Security Features**:
- **Address Space Layout Randomization (ASLR)**: `randomize_va_space` support
- **Memory Protection Keys**: Intel MPX support
- **Control Flow Integrity**: CET support
- **Memory Tagging**: ARM MTE integration

### Page Table Management

#### Page Table Allocation and Deallocation
```c
static void free_pte_range(struct mmu_gather *tlb, pmd_t *pmd,
                          unsigned long addr)
static inline void free_pmd_range(struct mmu_gather *tlb, pud_t *pud, ...)
static inline void free_pud_range(struct mmu_gather *tlb, p4d_t *p4d, ...)
static inline void free_p4d_range(struct mmu_gather *tlb, pgd_t *pgd, ...)
```

**Page Table Features**:
- **Hierarchical Management**: Four/five-level page table hierarchy
- **Lazy Allocation**: On-demand page table allocation
- **Efficient Deallocation**: Batch page table freeing
- **Memory Overhead Optimization**: Minimize page table memory usage

#### TLB Management Integration
- **TLB Invalidation**: Proper TLB invalidation coordination
- **Batch Operations**: Efficient batch TLB operations
- **Multi-processor Coordination**: SMP-safe TLB management
- **Performance Optimization**: Minimize TLB flush overhead

## Advanced Features

### Transparent Huge Pages (THP)

#### Huge Page Fault Handling
```c
/* Create huge PMD */
ret = create_huge_pmd(&vmf);
if (!(ret & VM_FAULT_FALLBACK))
    return ret;

/* Handle huge PMD write protection */
ret = wp_huge_pmd(&vmf);
if (!(ret & VM_FAULT_FALLBACK))
    return ret;
```

**THP Features**:
- **Automatic Promotion**: Automatic small page to huge page promotion
- **Fallback Handling**: Graceful fallback to regular pages
- **NUMA Integration**: NUMA-aware huge page management
- **Performance Benefits**: Reduced TLB pressure and page table overhead

### Memory Hotplug and Migration

#### Page Migration Integration
- **NUMA Migration**: Automatic NUMA balancing migration
- **Memory Compaction**: Integration with memory compaction
- **Hot-remove Support**: Memory hot-remove support
- **CMA Integration**: Contiguous Memory Allocator integration

### High Memory (HIGHMEM) Support

#### HIGHMEM Features
- **Kmap Integration**: Temporary kernel mapping support
- **Bounce Buffers**: DMA bounce buffer support
- **32-bit Optimization**: Optimization for 32-bit systems
- **Legacy Support**: Backward compatibility

### Performance Optimizations

#### Fast Path Optimizations
- **Lock-free Operations**: Lock-free page table reading
- **Speculative Operations**: Speculative page table operations
- **Cache-friendly Data Structures**: Optimized data structure layout
- **Branch Prediction**: Optimized for common cases

#### Memory Access Patterns
- **Readahead Integration**: Integration with page readahead
- **Prefaulting**: Speculative page prefaulting
- **Access Tracking**: Memory access pattern tracking
- **Hot/Cold Page Classification**: Page temperature tracking

## Error Handling and Recovery

### Fault Error Processing
- **OOM Handling**: Out-of-memory condition handling
- **Hardware Errors**: Memory hardware error handling
- **Signal Generation**: Appropriate signal generation (SIGSEGV, SIGBUS)
- **Error Propagation**: Proper error code propagation

### Memory Poisoning Integration
- **Hardware Poisoning**: Hardware memory error isolation
- **Software Poisoning**: Software-triggered memory poisoning
- **Recovery Mechanisms**: Error recovery and containment
- **Notification Systems**: Error notification to user space

## Integration Points

### File System Integration
- **File-backed Memory**: Memory-mapped file support
- **Page Cache**: Integration with page cache
- **Direct I/O**: Direct I/O memory management
- **Filesystem Fault Handlers**: Custom filesystem fault handling

### Device Driver Integration
- **DMA Mapping**: DMA buffer management
- **Device Memory**: Memory-mapped device support
- **GPU Memory**: Graphics memory management
- **Custom Memory Types**: Support for special memory types

### Container and Virtualization Integration
- **Memory Cgroups**: Memory control group integration
- **Virtual Memory**: Virtualization memory management
- **Memory Balloon**: Memory ballooning support
- **Live Migration**: VM live migration support

### Security Framework Integration
- **SELinux**: Security Enhanced Linux integration
- **SMACK**: Simplified Mandatory Access Control
- **Memory Protection**: Hardware memory protection features
- **Kernel Address Space Layout Randomization (KASLR)**: Kernel ASLR support

This comprehensive memory management implementation provides the foundation for virtual memory operations in Linux, enabling efficient, secure, and scalable memory management across diverse hardware platforms and workload requirements while maintaining compatibility with legacy applications and supporting modern features like huge pages, NUMA optimization, and advanced security mechanisms.