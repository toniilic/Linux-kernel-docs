# Linux Kernel Memory Management: mmap.c

## Overview

The `mmap.c` file implements **memory mapping** functionality for the Linux kernel, providing the infrastructure for mapping files and anonymous memory into process address spaces. This is the foundation for **Virtual Memory Area (VMA) management** and handles user-space memory allocation through the `mmap()` system call and related operations.

## Core Functionality

### 1. Memory Mapping Concepts

Memory mapping allows processes to:
- **Map files into memory**: Access file contents through memory operations
- **Create anonymous mappings**: Allocate memory regions not backed by files
- **Share memory**: Multiple processes can map the same memory region
- **Private mappings**: Copy-on-write semantics for private modifications

#### Key Benefits
- **Efficient I/O**: File access through memory operations
- **Memory sharing**: Inter-process communication and library sharing
- **Lazy loading**: Pages loaded on-demand during access
- **Virtual memory management**: Flexible address space layout

### 2. Virtual Memory Areas (VMAs)

VMAs are the fundamental unit of memory management:

```c
struct vm_area_struct {
    unsigned long vm_start;         // Start virtual address
    unsigned long vm_end;           // End virtual address (exclusive)
    struct mm_struct *vm_mm;        // Associated memory descriptor
    
    vm_flags_t vm_flags;           // Access permissions and properties
    pgprot_t vm_page_prot;         // Page protection bits
    
    struct file *vm_file;          // Mapped file (if any)
    unsigned long vm_pgoff;        // Offset into file (pages)
    
    const struct vm_operations_struct *vm_ops; // VMA-specific operations
    void *vm_private_data;         // Private data for VMA
    
    // Red-black tree and list linkage
    struct rb_node vm_rb;
    struct list_head anon_vma_chain;
};
```

#### VM Flags
- `VM_READ`: Pages may be read
- `VM_WRITE`: Pages may be written
- `VM_EXEC`: Pages may be executed
- `VM_SHARED`: Pages are shared
- `VM_MAYSHARE`: Flag affects whether pages may be shared
- `VM_GROWSDOWN`: Stack-like segment (grows downward)
- `VM_LOCKED`: Pages are locked in memory
- `VM_HUGETLB`: Use huge pages for this VMA

## System Call Interface

### 1. mmap() System Call

```c
SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
                unsigned long, prot, unsigned long, flags,
                unsigned long, fd, unsigned long, pgoff)
{
    return ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff);
}
```

#### Parameters
- **addr**: Hint for starting address (0 for kernel choice)
- **len**: Length of mapping in bytes
- **prot**: Memory protection (PROT_READ, PROT_WRITE, PROT_EXEC)
- **flags**: Mapping flags (MAP_SHARED, MAP_PRIVATE, MAP_ANONYMOUS, etc.)
- **fd**: File descriptor (for file-backed mappings)
- **pgoff**: Offset into file in pages

### 2. brk() System Call

```c
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
    struct mm_struct *mm = current->mm;
    unsigned long newbrk, oldbrk;
    
    // Validate new break point
    if (brk < mm->start_brk)
        return mm->brk;  // Return current break
    
    newbrk = PAGE_ALIGN(brk);
    oldbrk = PAGE_ALIGN(mm->brk);
    
    if (newbrk == oldbrk) {
        mm->brk = brk;
        return brk;
    }
    
    // Handle shrinking or growing heap
    if (brk <= mm->brk) {
        // Shrink heap
        return do_munmap(mm, newbrk, oldbrk - newbrk);
    } else {
        // Grow heap
        return do_brk_flags(oldbrk, newbrk - oldbrk, 0);
    }
}
```

## Core Mapping Functions

### 1. do_mmap() - Main Mapping Function

```c
unsigned long do_mmap(struct file *file, unsigned long addr,
                     unsigned long len, unsigned long prot,
                     unsigned long flags, vm_flags_t vm_flags,
                     unsigned long pgoff, unsigned long *populate,
                     struct list_head *uf)
{
    struct mm_struct *mm = current->mm;
    
    // Input validation
    if (!len)
        return -EINVAL;
    
    // Handle protection and personality flags
    if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
        if (!(file && path_noexec(&file->f_path)))
            prot |= PROT_EXEC;
    
    // Calculate VM flags from protection and mapping flags
    vm_flags |= calc_vm_prot_bits(prot, pkey) | 
                calc_vm_flag_bits(file, flags) |
                mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;
    
    // Find unmapped area
    addr = __get_unmapped_area(file, addr, len, pgoff, flags, vm_flags);
    if (IS_ERR_VALUE(addr))
        return addr;
    
    // Perform the actual mapping
    addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);
    
    // Set population requirement
    if (!IS_ERR_VALUE(addr) && 
        ((vm_flags & VM_LOCKED) || 
         (flags & (MAP_POPULATE | MAP_NONBLOCK)) == MAP_POPULATE))
        *populate = len;
    
    return addr;
}
```

### 2. mmap_region() - Core Mapping Implementation

```c
unsigned long mmap_region(struct file *file, unsigned long addr,
                         unsigned long len, vm_flags_t vm_flags, 
                         unsigned long pgoff, struct list_head *uf)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma = NULL;
    
    // Security checks
    if (map_deny_write_exec(vm_flags, vm_flags))
        return -EACCES;
    
    if (!arch_validate_flags(vm_flags))
        return -EINVAL;
    
    // Handle writable file mappings
    if (file && is_shared_maywrite(vm_flags)) {
        int error = mapping_map_writable(file->f_mapping);
        if (error)
            return error;
    }
    
    // Try to merge with existing VMAs
    vma = vma_merge_new_range(&vmg);
    
    // If merge failed, create new VMA
    if (!vma) {
        vma = vm_area_alloc(mm);
        if (!vma)
            return -ENOMEM;
        
        vma_set_range(vma, addr, addr + len, pgoff);
        vm_flags_init(vma, vm_flags);
        vma->vm_page_prot = vm_get_page_prot(vm_flags);
        
        if (file) {
            vma->vm_file = get_file(file);
            error = call_mmap(file, vma);
            if (error)
                goto free_vma;
        }
    }
    
    // Insert VMA into MM structures
    vma_iter_store(&vmi, vma);
    mm->map_count++;
    mm->total_vm += len >> PAGE_SHIFT;
    
    return addr;
}
```

## Address Space Management

### 1. Unmapped Area Search

```c
unsigned long unmapped_area(struct vm_unmapped_area_info *info)
{
    unsigned long length, gap;
    struct vm_area_struct *tmp;
    VMA_ITERATOR(vmi, current->mm, 0);
    
    // Adjust search length for alignment
    length = info->length + info->align_mask + info->start_gap;
    
    // Find lowest suitable area
    if (vma_iter_area_lowest(&vmi, info->low_limit, 
                           info->high_limit, length))
        return -ENOMEM;
    
    // Calculate aligned gap
    gap = vma_iter_addr(&vmi) + info->start_gap;
    gap += (info->align_offset - gap) & info->align_mask;
    
    // Check for conflicts with adjacent VMAs
    tmp = vma_next(&vmi);
    if (tmp && (tmp->vm_flags & VM_STARTGAP_FLAGS)) {
        if (vm_start_gap(tmp) < gap + length - 1) {
            // Retry search from higher address
            goto retry_higher;
        }
    }
    
    return gap;
}
```

### 2. Top-Down Area Search

```c
unsigned long unmapped_area_topdown(struct vm_unmapped_area_info *info)
{
    // Similar to unmapped_area() but searches from high to low addresses
    // Used when mm->get_unmapped_area == arch_get_unmapped_area_topdown
    
    if (vma_iter_area_highest(&vmi, info->low_limit, 
                            info->high_limit, length))
        return -ENOMEM;
    
    gap = vma_iter_end(&vmi) - info->length;
    gap -= (gap - info->align_offset) & info->align_mask;
    
    return gap;
}
```

## VMA Operations and Management

### 1. VMA Merging

```c
struct vm_area_struct *vma_merge_new_range(struct vma_merge_struct *vmg)
{
    struct vm_area_struct *prev = vmg->prev;
    struct vm_area_struct *next = vmg->next;
    
    // Check if we can merge with previous VMA
    if (prev && can_vma_merge_after(prev, vmg->flags, 
                                   vmg->anon_vma, vmg->file, 
                                   vmg->pgoff, vmg->policy)) {
        // Check if we can also merge with next VMA (three-way merge)
        if (next && can_vma_merge_before(next, vmg->flags,
                                       vmg->anon_vma, vmg->file,
                                       vmg->pgoff, vmg->policy)) {
            // Three-way merge: prev + new + next
            return vma_merge_three_way(vmg);
        } else {
            // Two-way merge: prev + new
            return vma_merge_expand_prev(vmg);
        }
    } else if (next && can_vma_merge_before(next, vmg->flags,
                                          vmg->anon_vma, vmg->file,
                                          vmg->pgoff, vmg->policy)) {
        // Two-way merge: new + next
        return vma_merge_expand_next(vmg);
    }
    
    // No merge possible
    return NULL;
}
```

### 2. VMA Splitting

```c
int split_vma(struct vma_iterator *vmi, struct vm_area_struct *vma,
             unsigned long addr, int new_below)
{
    struct vm_area_struct *new;
    
    // Validate split point
    if (vma->vm_start >= addr || vma->vm_end <= addr)
        return -EINVAL;
    
    // Allocate new VMA
    new = vm_area_dup(vma);
    if (!new)
        return -ENOMEM;
    
    if (new_below) {
        // Split: [orig_start, addr) and [addr, orig_end)
        new->vm_end = addr;
        vma->vm_start = addr;
        vma->vm_pgoff += (addr - new->vm_start) >> PAGE_SHIFT;
    } else {
        // Split: [orig_start, addr) and [addr, orig_end)
        new->vm_start = addr;
        new->vm_pgoff += (addr - vma->vm_start) >> PAGE_SHIFT;
        vma->vm_end = addr;
    }
    
    // Handle anon_vma and file references
    if (new->vm_file)
        get_file(new->vm_file);
    
    // Insert new VMA into data structures
    vma_iter_store(vmi, new);
    return 0;
}
```

## Memory Protection and Security

### 1. Protection Validation

```c
static bool arch_validate_flags(vm_flags_t vm_flags)
{
    // Architecture-specific validation
    // Check for incompatible flag combinations
    
    if ((vm_flags & VM_EXEC) && (vm_flags & VM_WRITE))
        if (map_deny_write_exec(vm_flags, vm_flags))
            return false;
    
    return arch_validate_prot(vm_flags);
}
```

### 2. ASLR (Address Space Layout Randomization)

```c
#ifdef CONFIG_HAVE_ARCH_MMAP_RND_BITS
static unsigned long mmap_rnd(void)
{
    unsigned long rnd = 0;
    
    if (current->flags & PF_RANDOMIZE) {
#ifdef CONFIG_COMPAT
        if (in_compat_syscall())
            rnd = get_random_long() & ((1UL << mmap_rnd_compat_bits) - 1);
        else
#endif
            rnd = get_random_long() & ((1UL << mmap_rnd_bits) - 1);
        
        rnd <<= PAGE_SHIFT;
    }
    return rnd;
}
#endif
```

### 3. Stack Guard Gaps

```c
static inline unsigned long vm_start_gap(struct vm_area_struct *vma)
{
    unsigned long vm_start = vma->vm_start;
    
    if (vma->vm_flags & VM_GROWSDOWN) {
        vm_start -= stack_guard_gap;
        if (vm_start > vma->vm_start)
            vm_start = 0;
    } else if (vma->vm_flags & VM_SHADOW_STACK) {
        vm_start -= PAGE_SIZE;
        if (vm_start > vma->vm_start)
            vm_start = 0;
    }
    
    return vm_start;
}
```

## File-Backed Mappings

### 1. File Mapping Setup

```c
static int call_mmap(struct file *file, struct vm_area_struct *vma)
{
    int error;
    
    // Security and capability checks
    error = security_mmap_file(file, vma->vm_prot, vma->vm_flags);
    if (error)
        return error;
    
    // Call file-system specific mmap handler
    if (file->f_op->mmap) {
        error = file->f_op->mmap(file, vma);
        if (error)
            return error;
    }
    
    // Set up file-specific operations
    if (vma->vm_ops && vma->vm_ops->open)
        vma->vm_ops->open(vma);
    
    return 0;
}
```

### 2. Shared Memory Validation

```c
static int file_mmap_ok(struct file *file, struct inode *inode,
                       unsigned long pgoff, unsigned long len)
{
    u64 maxsize = file_mmap_size_max(file, inode);
    
    if (maxsize && len > maxsize)
        return false;
    
    maxsize -= len;
    if (pgoff > maxsize >> PAGE_SHIFT)
        return false;
    
    return true;
}
```

## Anonymous Memory Management

### 1. Anonymous VMA Creation

```c
int do_brk_flags(struct vma_iterator *vmi, struct vm_area_struct *vma,
                unsigned long addr, unsigned long len, unsigned long flags)
{
    struct mm_struct *mm = current->mm;
    
    // Calculate flags for anonymous mapping
    flags |= VM_DATA_DEFAULT_FLAGS | VM_ACCOUNT | mm->def_flags;
    
    // Check memory limits
    if (!may_expand_vm(mm, flags, len >> PAGE_SHIFT))
        return -ENOMEM;
    
    if (security_vm_enough_memory_mm(mm, len >> PAGE_SHIFT))
        return -ENOMEM;
    
    // Try to expand existing VMA
    if (vma && vma->vm_end == addr && can_vma_merge_after(vma, flags)) {
        vma->vm_end += len;
        goto out;
    }
    
    // Create new anonymous VMA
    vma = vm_area_alloc(mm);
    if (!vma)
        return -ENOMEM;
    
    vma_set_anonymous(vma);
    vma_set_range(vma, addr, addr + len, addr >> PAGE_SHIFT);
    vm_flags_init(vma, flags);
    
    // Insert into MM structures
    vma_iter_store(vmi, vma);
    mm->map_count++;
    
out:
    mm->total_vm += len >> PAGE_SHIFT;
    mm->data_vm += len >> PAGE_SHIFT;
    if (flags & VM_LOCKED)
        mm->locked_vm += len >> PAGE_SHIFT;
    
    return 0;
}
```

## Performance Optimizations

### 1. VMA Caching

- **VMA lookup cache**: Recently accessed VMAs cached for faster lookup
- **Red-black tree**: O(log n) VMA lookup by address
- **Interval tree**: Efficient range-based VMA queries

### 2. Lazy Allocation

```c
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct page *page;
    
    // Allocate page only when accessed (demand paging)
    page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
    if (!page)
        return VM_FAULT_OOM;
    
    // Set up page table entry
    entry = mk_pte(page, vma->vm_page_prot);
    if (vma->vm_flags & VM_WRITE)
        entry = pte_mkwrite(pte_mkdirty(entry));
    
    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
    return 0;
}
```

### 3. Huge Page Support

```c
static bool transhuge_vma_suitable(struct vm_area_struct *vma,
                                  unsigned long addr)
{
    if (vma->vm_start > addr || vma->vm_end < addr + HPAGE_PMD_SIZE)
        return false;
    
    if (vma->vm_flags & VM_NOHUGEPAGE)
        return false;
    
    return true;
}
```

## Error Handling and Edge Cases

### 1. Resource Limits

- **RLIMIT_AS**: Total address space limit
- **RLIMIT_DATA**: Data segment size limit  
- **RLIMIT_MEMLOCK**: Memory locking limit
- **sysctl_max_map_count**: Maximum number of VMAs

### 2. MAP_FIXED Handling

```c
if (flags & MAP_FIXED) {
    // Fixed mappings must succeed at exact address
    if (addr & ~PAGE_MASK)
        return -EINVAL;
    
    // Unmap any existing mappings in range
    if (find_vma_intersection(mm, addr, addr + len)) {
        if (do_munmap(mm, addr, len, uf))
            return -ENOMEM;
    }
}
```

### 3. NUMA Policy Integration

```c
struct mempolicy *get_vma_policy(struct vm_area_struct *vma,
                               unsigned long addr, int order, pgoff_t *ilx)
{
    struct mempolicy *pol = NULL;
    
    if (vma) {
        if (vma->vm_ops && vma->vm_ops->get_policy)
            pol = vma->vm_ops->get_policy(vma, addr, ilx);
        else if (vma->vm_policy)
            pol = vma->vm_policy;
    }
    
    if (!pol)
        pol = &default_policy;
    
    return pol;
}
```

This mmap implementation provides the foundation for user-space memory management in Linux, handling everything from simple heap allocation via brk() to complex file mappings with sophisticated VMA merging and splitting algorithms.