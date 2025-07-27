# Linux Kernel Memory Management: vmalloc.c

## Overview

The `vmalloc.c` file implements **virtual memory allocation** for the Linux kernel, providing **virtually contiguous memory regions** that may be physically discontiguous. This is essential for allocating large memory areas in kernel space without requiring physically contiguous pages, which become increasingly scarce as the system runs.

## Core Functionality

### 1. Virtual Memory Allocation

Unlike kmalloc which provides both virtual and physical contiguity, vmalloc provides:
- **Virtual contiguity**: Consecutive virtual addresses
- **Physical discontiguity**: Pages can be scattered in physical memory
- **Large allocations**: Handles multi-megabyte allocations efficiently
- **Page-based mapping**: Uses page table mappings for virtual-to-physical translation

#### Key Benefits
- **Fragmentation resistance**: Works even when physical memory is fragmented
- **Large allocation support**: Can allocate memory larger than available contiguous physical memory
- **NUMA awareness**: Can allocate from specific NUMA nodes
- **Flexible protection**: Supports different page protection attributes

### 2. Virtual Address Space Management

Vmalloc operates in a dedicated region of kernel virtual address space:

```c
// Virtual address space layout (architecture-dependent)
#define VMALLOC_START   /* Start of vmalloc area */
#define VMALLOC_END     /* End of vmalloc area */

bool is_vmalloc_addr(const void *x)
{
    unsigned long addr = (unsigned long)kasan_reset_tag(x);
    return addr >= VMALLOC_START && addr < VMALLOC_END;
}
```

## Key Data Structures

### 1. VM Area Structure

```c
struct vm_struct {
    void *addr;                     // Virtual address of the area
    unsigned long size;             // Size of the allocated area
    unsigned long flags;            // Allocation flags (VM_ALLOC, VM_IOREMAP, etc.)
    struct page **pages;            // Array of backing physical pages
    unsigned int nr_pages;          // Number of pages in the area
    phys_addr_t phys_addr;         // Physical address (for ioremap)
    const void *caller;            // Caller function for debugging
    struct vm_struct *next;        // Linked list pointer (legacy)
    unsigned long requested_size;   // Originally requested size
};
```

#### VM Flags
- `VM_ALLOC`: General vmalloc allocation
- `VM_IOREMAP`: I/O memory mapping
- `VM_MAP`: Map existing pages
- `VM_UNINITIALIZED`: Area not yet fully initialized
- `VM_NO_GUARD`: No guard page at end
- `VM_ALLOW_HUGE_VMAP`: Allow huge page mappings

### 2. VMap Area Structure

```c
struct vmap_area {
    unsigned long va_start;         // Start virtual address
    unsigned long va_end;           // End virtual address
    struct rb_node rb_node;         // Red-black tree node
    struct list_head list;          // Free/used list
    union {
        unsigned long subtree_max_size; // For allocation search
        struct vm_struct *vm;       // Associated vm_struct
    };
    unsigned long flags;            // Area flags
};
```

### 3. Deferred Free Structure

```c
struct vfree_deferred {
    struct llist_head list;         // List of areas to free
    struct work_struct wq;          // Work queue for deferred processing
};
```

## Page Table Management

### 1. Multi-Level Page Table Creation

Vmalloc creates page table entries across all levels:

```c
// Page table hierarchy for vmalloc mappings
static int vmap_range_noflush(unsigned long addr, unsigned long end,
                             phys_addr_t phys_addr, pgprot_t prot,
                             unsigned int max_page_shift)
{
    pgd_t *pgd;
    // Walk through all page table levels:
    // PGD → P4D → PUD → PMD → PTE
    
    do {
        next = pgd_addr_end(addr, end);
        err = vmap_p4d_range(pgd, addr, next, phys_addr, prot,
                           max_page_shift, &mask);
    } while (pgd++, phys_addr += (next - addr), addr = next, addr != end);
    
    return err;
}
```

### 2. Huge Page Support

```c
#ifdef CONFIG_HAVE_ARCH_HUGE_VMAP
// Try to use huge pages for better TLB efficiency
static int vmap_try_huge_pmd(pmd_t *pmd, unsigned long addr, unsigned long end,
                           phys_addr_t phys_addr, pgprot_t prot,
                           unsigned int max_page_shift)
{
    if (max_page_shift < PMD_SHIFT)
        return 0;
    
    if ((end - addr) != PMD_SIZE)
        return 0;
    
    if (!IS_ALIGNED(addr, PMD_SIZE) || !IS_ALIGNED(phys_addr, PMD_SIZE))
        return 0;
    
    return pmd_set_huge(pmd, phys_addr, prot);
}
#endif
```

### 3. Page Table Cleanup

```c
void __vunmap_range_noflush(unsigned long start, unsigned long end)
{
    // Remove page table entries at all levels
    // Handle huge pages appropriately
    // Update page table modification masks
    
    do {
        next = pgd_addr_end(addr, end);
        vunmap_p4d_range(pgd, addr, next, &mask);
    } while (pgd++, addr = next, addr != end);
    
    // Synchronize architecture-specific page tables
    if (mask & ARCH_PAGE_TABLE_SYNC_MASK)
        arch_sync_kernel_mappings(start, end);
}
```

## Allocation Algorithms

### 1. Area Allocation (`__get_vm_area_node`)

```c
struct vm_struct *__get_vm_area_node(unsigned long size,
        unsigned long align, unsigned long shift, unsigned long flags,
        unsigned long start, unsigned long end, int node,
        gfp_t gfp_mask, const void *caller)
{
    struct vmap_area *va;
    struct vm_struct *area;
    
    // Allocate vm_struct descriptor
    area = kzalloc_node(sizeof(*area), gfp_mask & GFP_RECLAIM_MASK, node);
    if (unlikely(!area))
        return NULL;
    
    // Add guard page unless explicitly disabled
    if (!(flags & VM_NO_GUARD))
        size += PAGE_SIZE;
    
    // Find free virtual address space
    va = alloc_vmap_area(size, align, start, end, node, gfp_mask, 0, area);
    if (IS_ERR(va)) {
        kfree(area);
        return NULL;
    }
    
    return area;
}
```

### 2. Physical Page Allocation and Mapping

```c
static void *__vmalloc_area_node(struct vm_struct *area, gfp_t gfp_mask,
                                pgprot_t prot, unsigned int page_shift,
                                int node)
{
    unsigned long addr = (unsigned long)area->addr;
    unsigned long size = get_vm_area_size(area);
    unsigned int nr_small_pages = size >> PAGE_SHIFT;
    
    // Allocate page pointer array
    if (array_size > PAGE_SIZE) {
        // Large arrays use vmalloc recursively
        area->pages = __vmalloc_node_noprof(array_size, 1, nested_gfp, 
                                          node, area->caller);
    } else {
        area->pages = kmalloc_node_noprof(array_size, nested_gfp, node);
    }
    
    // Allocate physical pages
    area->nr_pages = vm_area_alloc_pages(gfp_mask, node, page_order,
                                       nr_small_pages, area->pages);
    
    // Map pages into virtual address space
    ret = vmap_pages_range(addr, addr + size, prot, area->pages, page_shift);
    if (ret < 0)
        goto fail;
    
    return area->addr;
}
```

### 3. Main Allocation Interface

```c
void *__vmalloc_node_range_noprof(unsigned long size, unsigned long align,
        unsigned long start, unsigned long end, gfp_t gfp_mask,
        pgprot_t prot, unsigned long vm_flags, int node,
        const void *caller)
{
    struct vm_struct *area;
    void *ret;
    
    // Size validation
    if (WARN_ON_ONCE(!size))
        return NULL;
    
    if ((size >> PAGE_SHIFT) > totalram_pages()) {
        warn_alloc(gfp_mask, NULL, "vmalloc error: size %lu, exceeds total pages", size);
        return NULL;
    }
    
    // Try huge pages if enabled and appropriate
    if (vmap_allow_huge && (vm_flags & VM_ALLOW_HUGE_VMAP)) {
        if (arch_vmap_pmd_supported(prot) && size >= PMD_SIZE)
            shift = PMD_SHIFT;
        align = max(original_align, 1UL << shift);
    }
    
again:
    // Allocate virtual area
    area = __get_vm_area_node(size, align, shift, VM_ALLOC | VM_UNINITIALIZED | vm_flags,
                            start, end, node, gfp_mask, caller);
    if (!area) {
        if (gfp_mask & __GFP_NOFAIL) {
            schedule_timeout_uninterruptible(1);
            goto again;
        }
        return NULL;
    }
    
    // Allocate and map physical pages
    ret = __vmalloc_area_node(area, gfp_mask, prot, shift, node);
    if (!ret)
        goto fail;
    
    // Security and debugging hooks
    area->addr = kasan_unpoison_vmalloc(area->addr, size, kasan_flags);
    clear_vm_uninitialized_flag(area);
    kmemleak_vmalloc(area, PAGE_ALIGN(size), gfp_mask);
    
    return area->addr;
}
```

## Deallocation Process

### 1. Virtual Area Removal

```c
struct vm_struct *remove_vm_area(const void *addr)
{
    struct vmap_area *va;
    struct vm_struct *vm;
    
    // Find and unlink vmap area
    va = find_unlink_vmap_area((unsigned long)addr);
    if (!va || !va->vm)
        return NULL;
    
    vm = va->vm;
    
    // Security and debugging cleanup
    debug_check_no_locks_freed(vm->addr, get_vm_area_size(vm));
    debug_check_no_obj_freed(vm->addr, get_vm_area_size(vm));
    kasan_free_module_shadow(vm);
    kasan_poison_vmalloc(vm->addr, get_vm_area_size(vm));
    
    free_unmap_vmap_area(va);
    return vm;
}
```

### 2. Deferred Freeing

```c
// Per-CPU deferred free to avoid expensive operations in interrupt context
static DEFINE_PER_CPU(struct vfree_deferred, vfree_deferred);

static void delayed_vfree_work(struct work_struct *w)
{
    struct vfree_deferred *p = container_of(w, struct vfree_deferred, wq);
    struct llist_node *t, *llnode;
    
    // Process deferred free list
    llnode = llist_del_all(&p->list);
    llist_for_each_safe(llnode, t, llnode)
        vfree(llnode);
}

void vfree(const void *addr)
{
    if (unlikely(in_interrupt())) {
        // Defer freeing in interrupt context
        struct vfree_deferred *p = this_cpu_ptr(&vfree_deferred);
        llist_add(&area->rcu_head, &p->list);
        schedule_work(&p->wq);
    } else {
        // Direct freeing in process context
        __vfree(addr);
    }
}
```

## Performance Optimizations

### 1. Huge Page Support

```c
#ifdef CONFIG_HAVE_ARCH_HUGE_VMALLOC
// Use large pages to reduce TLB pressure
if (vmap_allow_huge && (vm_flags & VM_ALLOW_HUGE_VMAP)) {
    // PMD-level mappings (2MB on x86-64)
    if (arch_vmap_pmd_supported(prot) && size >= PMD_SIZE)
        shift = PMD_SHIFT;
    // PUD-level mappings (1GB on x86-64)  
    else if (arch_vmap_pud_supported(prot) && size >= PUD_SIZE)
        shift = PUD_SHIFT;
}
#endif
```

### 2. NUMA Awareness

```c
// Allocate from specific NUMA node for locality
void *__vmalloc_node_noprof(unsigned long size, unsigned long align,
                           gfp_t gfp_mask, int node, const void *caller)
{
    return __vmalloc_node_range_noprof(size, align, VMALLOC_START, VMALLOC_END,
                                     gfp_mask, PAGE_KERNEL, 0, node, caller);
}
```

### 3. Page Order Optimization

```c
// Try higher-order pages first for efficiency
area->nr_pages = vm_area_alloc_pages(
    (page_order ? gfp_mask & ~__GFP_NOFAIL : gfp_mask) | __GFP_NOWARN,
    node, page_order, nr_small_pages, area->pages);

// Fallback to order-0 pages if high-order allocation fails
if (area->nr_pages != nr_small_pages && page_order > 0) {
    // Retry with order-0 pages
}
```

## Security Features

### 1. Guard Pages

```c
// Add guard page to detect buffer overflows
if (!(flags & VM_NO_GUARD))
    size += PAGE_SIZE;  // Add guard page at end
```

### 2. Address Space Layout Randomization

- Virtual address allocation includes randomization
- Makes exploitation more difficult
- Configurable through KASLR settings

### 3. Permission Management

```c
static void vm_reset_perms(struct vm_struct *area)
{
    // Reset direct map permissions for security
    set_area_direct_map(area, set_direct_map_invalid_noflush);
    _vm_unmap_aliases(start, end, flush_dmap);
    set_area_direct_map(area, set_direct_map_default_noflush);
}
```

## Integration Points

### 1. I/O Memory Mapping

```c
void __iomem *ioremap(phys_addr_t phys_addr, size_t size)
{
    // Map physical device memory into kernel virtual space
    return __ioremap_caller(phys_addr, size, PAGE_KERNEL_IO,
                          __builtin_return_address(0), false);
}
```

### 2. Module Loading

- Module code and data sections use vmalloc
- Supports executable permissions for code sections
- Handles module unloading and cleanup

### 3. Memory Control Groups

```c
// Account vmalloc allocations to memory cgroups
if (gfp_mask & __GFP_ACCOUNT && area->nr_pages)
    mod_memcg_page_state(area->pages[0], MEMCG_VMALLOC, area->nr_pages);
```

## Error Handling and Debugging

### 1. Allocation Failure Handling

```c
if (area->nr_pages != nr_small_pages) {
    if (!fatal_signal_pending(current) && page_order == 0)
        warn_alloc(gfp_mask, NULL,
                  "vmalloc error: size %lu, failed to allocate pages",
                  area->nr_pages * PAGE_SIZE);
    goto fail;
}
```

### 2. Debug Information

```c
static int vmalloc_info_show(struct seq_file *m, void *p)
{
    struct vm_struct *v = p;
    
    seq_printf(m, "0x%pK-0x%pK %7ld", 
              v->addr, v->addr + v->size, v->size);
    
    if (v->caller)
        seq_printf(m, " %pS\n", v->caller);
    else
        seq_putc(m, '\n');
    
    return 0;
}
```

### 3. Validation and Consistency Checks

- Page alignment validation
- Virtual address range checking
- Page table consistency verification
- Memory leak detection integration

## Common Use Cases

### 1. Large Kernel Data Structures

```c
// Allocate large arrays or buffers
void *large_buffer = vmalloc(LARGE_SIZE);
if (!large_buffer)
    return -ENOMEM;
```

### 2. Module Memory

```c
// Allocate executable memory for modules
void *module_mem = __vmalloc_node_range(size, 1, start, end,
                                       GFP_KERNEL, PAGE_KERNEL_EXEC,
                                       VM_FLUSH_RESET_PERMS, node, caller);
```

### 3. Device Driver Mappings

```c
// Map device registers
void __iomem *regs = ioremap(phys_base, reg_size);
if (!regs)
    return -EIO;
```

## Performance Characteristics

### Time Complexity

- **Allocation**: O(log n) for address space search
- **Mapping**: O(pages) for page table creation
- **Deallocation**: O(pages) for cleanup
- **TLB overhead**: Mitigated by huge page support

### Space Complexity

- **Virtual address space**: Limited by VMALLOC_START/END range
- **Page table overhead**: ~0.2% for 4KB pages, less for huge pages
- **Metadata overhead**: Small per-allocation vm_struct

### Fragmentation Characteristics

- **Virtual fragmentation**: Managed by red-black tree allocation
- **Physical fragmentation**: Irrelevant due to page table mapping
- **TLB fragmentation**: Reduced by huge page support

Vmalloc provides essential virtual memory services for the kernel, enabling large allocations and flexible memory management while maintaining good performance and security properties.