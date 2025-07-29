# Linux Kernel Cryptographic Core API (`crypto/api.c`)

## Overview

The `crypto/api.c` file implements the core infrastructure of Linux's Scatterlist Cryptographic API, providing the fundamental framework for algorithm registration, lookup, instantiation, and lifecycle management. This subsystem serves as the central coordinator for all cryptographic operations in the kernel, managing algorithm discovery, module loading, reference counting, and transform allocation. It implements sophisticated larval-based algorithm probing, FIPS compliance checking, and provides the foundation upon which all other cryptographic subsystems are built.

## Core Architecture

### 1. Global Algorithm Management

**Algorithm Registry** - Lines 26-32:
```c
LIST_HEAD(crypto_alg_list);
EXPORT_SYMBOL_GPL(crypto_alg_list);
DECLARE_RWSEM(crypto_alg_sem);
EXPORT_SYMBOL_GPL(crypto_alg_sem);

BLOCKING_NOTIFIER_HEAD(crypto_chain);
EXPORT_SYMBOL_GPL(crypto_chain);
```

**Global Infrastructure**:
- **Algorithm List**: Central registry of all available cryptographic algorithms
- **Read-Write Semaphore**: Protects algorithm list during modifications
- **Notifier Chain**: Coordinates algorithm registration and testing events
- **Boot Testing**: Manages algorithm validation during system initialization

### 2. Algorithm Reference Management

**Module Reference Counting** - Lines 43-56:
```c
struct crypto_alg *crypto_mod_get(struct crypto_alg *alg) {
    return try_module_get(alg->cra_module) ? crypto_alg_get(alg) : NULL;
}

void crypto_mod_put(struct crypto_alg *alg) {
    struct module *module = alg->cra_module;
    
    crypto_alg_put(alg);
    module_put(module);
}
```

**Reference Features**:
- **Module Safety**: Prevents module unloading while algorithms are in use
- **Algorithm References**: Maintains algorithm-level reference counts
- **Atomic Operations**: Uses atomic reference counting for thread safety
- **Proper Ordering**: Ensures correct module/algorithm reference ordering

### 3. Algorithm Lookup Infrastructure

**Core Lookup Algorithm** - Lines 58-91:
```c
static struct crypto_alg *__crypto_alg_lookup(const char *name, u32 type,
                                             u32 mask) {
    struct crypto_alg *q, *alg = NULL;
    int best = -2;
    
    list_for_each_entry(q, &crypto_alg_list, cra_list) {
        int exact, fuzzy;
        
        if (crypto_is_moribund(q))
            continue;
            
        if ((q->cra_flags ^ type) & mask)
            continue;
            
        exact = !strcmp(q->cra_driver_name, name);
        fuzzy = !strcmp(q->cra_name, name);
        if (!exact && !(fuzzy && q->cra_priority > best))
            continue;
            
        if (unlikely(!crypto_mod_get(q)))
            continue;
            
        best = q->cra_priority;
        if (alg)
            crypto_mod_put(alg);
        alg = q;
        
        if (exact)
            break;
    }
    
    return alg;
}
```

**Lookup Features**:
- **Priority-Based Selection**: Chooses highest priority algorithm when multiple matches exist
- **Exact vs Fuzzy Matching**: Driver name takes precedence over algorithm name
- **State Filtering**: Excludes dying or incompatible algorithms
- **Reference Management**: Acquires module references atomically
- **Type Mask Filtering**: Supports flexible algorithm type matching

## Larval-Based Algorithm Probing

### 1. Larval Algorithm Concept

**Larval Allocation** - Lines 103-123:
```c
struct crypto_larval *crypto_larval_alloc(const char *name, u32 type, u32 mask) {
    struct crypto_larval *larval;
    
    larval = kzalloc(sizeof(*larval), GFP_KERNEL);
    if (!larval)
        return ERR_PTR(-ENOMEM);
        
    type &= ~CRYPTO_ALG_TYPE_MASK | (mask ?: CRYPTO_ALG_TYPE_MASK);
    
    larval->mask = mask;
    larval->alg.cra_flags = CRYPTO_ALG_LARVAL | type;
    larval->alg.cra_priority = -1;
    larval->alg.cra_destroy = crypto_larval_destroy;
    
    strscpy(larval->alg.cra_name, name, CRYPTO_MAX_ALG_NAME);
    init_completion(&larval->completion);
    
    return larval;
}
```

**Larval Characteristics**:
- **Placeholder Algorithm**: Acts as temporary placeholder during algorithm construction
- **Completion Synchronization**: Uses completion mechanism for waiting threads
- **State Tracking**: Maintains test status and adult algorithm pointer
- **Reference Counting**: Proper cleanup when larval becomes unnecessary
- **Priority Handling**: Low priority to prefer real algorithms

### 2. Larval Lifecycle Management

**Larval Addition** - Lines 125-152:
```c
static struct crypto_alg *crypto_larval_add(const char *name, u32 type,
                                           u32 mask) {
    struct crypto_alg *alg;
    struct crypto_larval *larval;
    
    larval = crypto_larval_alloc(name, type, mask);
    if (IS_ERR(larval))
        return ERR_CAST(larval);
        
    refcount_set(&larval->alg.cra_refcnt, 2);
    
    down_write(&crypto_alg_sem);
    alg = __crypto_alg_lookup(name, type, mask);
    if (!alg) {
        alg = &larval->alg;
        list_add(&alg->cra_list, &crypto_alg_list);
    }
    up_write(&crypto_alg_sem);
    
    if (alg != &larval->alg) {
        kfree(larval);
        if (crypto_is_larval(alg))
            alg = crypto_larval_wait(alg, type, mask);
    }
    
    return alg;
}
```

**Lifecycle Features**:
- **Race Prevention**: Double-checks algorithm existence under write lock
- **Reference Management**: Starts with reference count of 2 for proper cleanup
- **Chaining Support**: Handles multiple concurrent larval requests
- **Memory Management**: Proper cleanup when larval becomes unnecessary

### 3. Larval Waiting and Resolution

**Larval Wait Protocol** - Lines 200-250:
```c
static struct crypto_alg *crypto_larval_wait(struct crypto_alg *alg,
                                           u32 type, u32 mask) {
    struct crypto_larval *larval;
    long time_left;
    
again:
    larval = container_of(alg, struct crypto_larval, alg);
    
    if (!crypto_boot_test_finished())
        crypto_start_test(larval);
        
    time_left = wait_for_completion_killable_timeout(
        &larval->completion, 60 * HZ);
        
    alg = larval->adult;
    if (time_left < 0)
        alg = ERR_PTR(-EINTR);
    else if (!time_left) {
        if (crypto_is_test_larval(larval))
            crypto_larval_kill(larval);
        alg = ERR_PTR(-ETIMEDOUT);
    } else if (!alg || PTR_ERR(alg) == -EEXIST) {
        int err = alg ? -EEXIST : -EAGAIN;
        
        alg = &larval->alg;
        alg = crypto_alg_lookup(alg->cra_name, type, mask) ?:
              ERR_PTR(err);
    } else if (IS_ERR(alg))
        ;
    else if (crypto_is_test_larval(larval) &&
             !(alg->cra_flags & CRYPTO_ALG_TESTED))
        alg = ERR_PTR(-EAGAIN);
    else if (alg->cra_flags & CRYPTO_ALG_FIPS_INTERNAL)
        alg = ERR_PTR(-EAGAIN);
    else if (!crypto_mod_get(alg))
        alg = ERR_PTR(-EAGAIN);
    crypto_mod_put(&larval->alg);
    
    if (!IS_ERR(alg) && crypto_is_larval(alg))
        goto again;
        
    return alg;
}
```

**Wait Protocol Features**:
- **Timeout Management**: 60-second timeout for algorithm construction
- **Signal Handling**: Killable wait allows interruption
- **Test Coordination**: Initiates algorithm testing during boot
- **State Validation**: Checks FIPS compliance and test status
- **Retry Logic**: Handles various failure scenarios with appropriate retries
- **Chain Resolution**: Resolves chains of larval algorithms

## Algorithm Discovery and Loading

### 1. Algorithm Lookup with Module Loading

**Larval Lookup with Module Loading** - Lines 289-321:
```c
static struct crypto_alg *crypto_larval_lookup(const char *name, u32 type,
                                              u32 mask) {
    struct crypto_alg *alg;
    
    if (!name)
        return ERR_PTR(-ENOENT);
        
    type &= ~(CRYPTO_ALG_LARVAL | CRYPTO_ALG_DEAD);
    mask &= ~(CRYPTO_ALG_LARVAL | CRYPTO_ALG_DEAD);
    
    alg = crypto_alg_lookup(name, type, mask);
    if (!alg && !(mask & CRYPTO_NOLOAD)) {
        request_module("crypto-%s", name);
        
        if (!((type ^ CRYPTO_ALG_NEED_FALLBACK) & mask &
              CRYPTO_ALG_NEED_FALLBACK))
            request_module("crypto-%s-all", name);
            
        alg = crypto_alg_lookup(name, type, mask);
    }
    
    if (!IS_ERR_OR_NULL(alg) && crypto_is_larval(alg))
        alg = crypto_larval_wait(alg, type, mask);
    else if (alg)
        ;
    else if (!(mask & CRYPTO_ALG_TESTED))
        alg = crypto_larval_add(name, type, mask);
    else
        alg = ERR_PTR(-ENOENT);
        
    return alg;
}
```

**Discovery Features**:
- **Module Auto-loading**: Attempts to load modules when algorithms not found
- **Fallback Support**: Loads fallback implementations when needed
- **State Filtering**: Excludes larval and dead algorithms from type matching
- **Conditional Loading**: Respects CRYPTO_NOLOAD flag to disable module loading
- **Test Integration**: Creates test larvals when algorithms need validation

### 2. FIPS Compliance and Testing

**FIPS-Aware Lookup** - Lines 252-287:
```c
static struct crypto_alg *crypto_alg_lookup(const char *name, u32 type,
                                           u32 mask) {
    const u32 fips = CRYPTO_ALG_FIPS_INTERNAL;
    struct crypto_alg *alg;
    u32 test = 0;
    
    if (!((type | mask) & CRYPTO_ALG_TESTED))
        test |= CRYPTO_ALG_TESTED;
        
    down_read(&crypto_alg_sem);
    alg = __crypto_alg_lookup(name, (type | test) & ~fips,
                             (mask | test) & ~fips);
    if (alg) {
        if (((type | mask) ^ fips) & fips)
            mask |= fips;
        mask &= fips;
        
        if (!crypto_is_larval(alg) &&
            ((type ^ alg->cra_flags) & mask)) {
            /* Algorithm is disallowed in FIPS mode. */
            crypto_mod_put(alg);
            alg = ERR_PTR(-ENOENT);
        }
    } else if (test) {
        alg = __crypto_alg_lookup(name, type, mask);
        if (alg && !crypto_is_larval(alg)) {
            /* Test failed */
            crypto_mod_put(alg);
            alg = ERR_PTR(-ELIBBAD);
        }
    }
    up_read(&crypto_alg_sem);
    
    return alg;
}
```

**FIPS Features**:
- **FIPS Mode Support**: Filters algorithms based on FIPS compliance requirements
- **Test Status Checking**: Ensures algorithms have passed required tests
- **Internal Algorithm Handling**: Manages FIPS_INTERNAL flagged algorithms
- **Test Failure Detection**: Returns ELIBBAD for algorithms that failed testing
- **Compliance Enforcement**: Prevents use of non-compliant algorithms in FIPS mode

## Transform Allocation and Management

### 1. Transform Memory Management

**Transform Allocation** - Lines 407-437:
```c
struct crypto_tfm *__crypto_alloc_tfmgfp(struct crypto_alg *alg, u32 type,
                                         u32 mask, gfp_t gfp) {
    struct crypto_tfm *tfm;
    unsigned int tfm_size;
    int err = -ENOMEM;
    
    tfm_size = sizeof(*tfm) + crypto_ctxsize(alg, type, mask);
    tfm = kzalloc(tfm_size, gfp);
    if (tfm == NULL)
        goto out_err;
        
    tfm->__crt_alg = alg;
    refcount_set(&tfm->refcnt, 1);
    
    if (!tfm->exit && alg->cra_init && (err = alg->cra_init(tfm)))
        goto cra_init_failed;
        
    goto out;
    
cra_init_failed:
    crypto_exit_ops(tfm);
    if (err == -EAGAIN)
        crypto_shoot_alg(alg);
    kfree(tfm);
out_err:
    tfm = ERR_PTR(err);
out:
    return tfm;
}
```

**Allocation Features**:
- **Context Size Calculation**: Dynamically calculates required context size
- **GFP Flag Support**: Allows specification of memory allocation flags
- **Initialization Chain**: Calls algorithm-specific initialization functions
- **Error Handling**: Proper cleanup on initialization failures
- **Algorithm Marking**: Marks algorithms as problematic on EAGAIN errors
- **Reference Counting**: Initializes transform reference count

### 2. Modern Transform Creation

**Frontend-Based Transform Creation** - Lines 526-560:
```c
void *crypto_create_tfm_node(struct crypto_alg *alg,
                             const struct crypto_type *frontend,
                             int node) {
    struct crypto_tfm *tfm;
    char *mem;
    int err;
    
    mem = crypto_alloc_tfmmem(alg, frontend, node, GFP_KERNEL);
    if (IS_ERR(mem))
        goto out;
        
    tfm = (struct crypto_tfm *)(mem + frontend->tfmsize);
    tfm->fb = tfm;
    
    err = frontend->init_tfm(tfm);
    if (err)
        goto out_free_tfm;
        
    if (!tfm->exit && alg->cra_init && (err = alg->cra_init(tfm)))
        goto cra_init_failed;
        
    goto out;
    
cra_init_failed:
    crypto_exit_ops(tfm);
out_free_tfm:
    if (err == -EAGAIN)
        crypto_shoot_alg(alg);
    kfree(mem);
    mem = ERR_PTR(err);
out:
    return mem;
}
```

**Frontend Features**:
- **NUMA Awareness**: Allocates on specified NUMA node
- **Frontend Integration**: Uses crypto_type structure for type-specific handling
- **Flexible Layout**: Supports frontend-specific memory layouts
- **Two-Stage Initialization**: Frontend init followed by algorithm init
- **Fallback Support**: Sets up fallback mechanism for transforms
- **Error Recovery**: Comprehensive error handling and cleanup

### 3. Transform Cloning

**Transform Cloning** - Lines 562-586:
```c
void *crypto_clone_tfm(const struct crypto_type *frontend,
                      struct crypto_tfm *otfm) {
    struct crypto_alg *alg = otfm->__crt_alg;
    struct crypto_tfm *tfm;
    char *mem;
    
    mem = ERR_PTR(-ESTALE);
    if (unlikely(!crypto_mod_get(alg)))
        goto out;
        
    mem = crypto_alloc_tfmmem(alg, frontend, otfm->node, GFP_ATOMIC);
    if (IS_ERR(mem)) {
        crypto_mod_put(alg);
        goto out;
    }
    
    tfm = (struct crypto_tfm *)(mem + frontend->tfmsize);
    tfm->crt_flags = otfm->crt_flags;
    tfm->fb = tfm;
    
out:
    return mem;
}
```

**Cloning Features**:
- **Atomic Allocation**: Uses GFP_ATOMIC for performance-critical cloning
- **Reference Safety**: Checks algorithm availability before cloning
- **State Preservation**: Copies relevant flags from original transform
- **NUMA Consistency**: Maintains same NUMA node as original
- **Lightweight Operation**: Minimal initialization for cloned transforms

## High-Level Allocation Interface

### 1. Base Transform Allocation

**Base Allocation Function** - Lines 468-500:
```c
struct crypto_tfm *crypto_alloc_base(const char *alg_name, u32 type, u32 mask) {
    struct crypto_tfm *tfm;
    int err;
    
    for (;;) {
        struct crypto_alg *alg;
        
        alg = crypto_alg_mod_lookup(alg_name, type, mask);
        if (IS_ERR(alg)) {
            err = PTR_ERR(alg);
            goto err;
        }
        
        tfm = __crypto_alloc_tfm(alg, type, mask);
        if (!IS_ERR(tfm))
            return tfm;
            
        crypto_mod_put(alg);
        err = PTR_ERR(tfm);
        
err:
        if (err != -EAGAIN)
            break;
        if (fatal_signal_pending(current)) {
            err = -EINTR;
            break;
        }
    }
    
    return ERR_PTR(err);
}
```

**Allocation Features**:
- **Retry Logic**: Handles EAGAIN errors with retry mechanism
- **Signal Awareness**: Respects fatal signals to avoid infinite loops
- **Algorithm Discovery**: Integrates module loading and larval resolution
- **Reference Management**: Proper algorithm reference handling
- **Error Propagation**: Maintains specific error codes for caller diagnosis

### 2. Modern NUMA-Aware Allocation

**NUMA-Aware Transform Allocation** - Lines 626-660:
```c
void *crypto_alloc_tfm_node(const char *alg_name,
                    const struct crypto_type *frontend, u32 type, u32 mask,
                    int node) {
    void *tfm;
    int err;
    
    for (;;) {
        struct crypto_alg *alg;
        
        alg = crypto_find_alg(alg_name, frontend, type, mask);
        if (IS_ERR(alg)) {
            err = PTR_ERR(alg);
            goto err;
        }
        
        tfm = crypto_create_tfm_node(alg, frontend, node);
        if (!IS_ERR(tfm))
            return tfm;
            
        crypto_mod_put(alg);
        err = PTR_ERR(tfm);
        
err:
        if (err != -EAGAIN)
            break;
        if (fatal_signal_pending(current)) {
            err = -EINTR;
            break;
        }
    }
    
    return ERR_PTR(err);
}
```

**Modern Features**:
- **Frontend Integration**: Works with modern crypto_type frontend system
- **NUMA Support**: Allows specification of preferred NUMA node
- **Type System**: Uses new type system with frontend-specific handling
- **Scalable Architecture**: Designed for multi-node systems
- **Performance Optimization**: Node-local allocation for better performance

## Utility Functions and Infrastructure

### 1. Algorithm Availability Checking

**Algorithm Existence Check** - Lines 689-701:
```c
int crypto_has_alg(const char *name, u32 type, u32 mask) {
    int ret = 0;
    struct crypto_alg *alg = crypto_alg_mod_lookup(name, type, mask);
    
    if (!IS_ERR(alg)) {
        crypto_mod_put(alg);
        ret = 1;
    }
    
    return ret;
}
```

**Utility Features**:
- **Non-allocating Check**: Determines algorithm availability without allocation
- **Module Loading**: Triggers module loading if algorithm not found
- **Reference Safety**: Properly releases acquired references
- **Simple Interface**: Boolean return for easy conditional logic

### 2. Asynchronous Request Infrastructure

**Request Completion Handling** - Lines 703-713:
```c
void crypto_req_done(void *data, int err) {
    struct crypto_wait *wait = data;
    
    if (err == -EINPROGRESS)
        return;
        
    wait->err = err;
    complete(&wait->completion);
}
```

**Async Features**:
- **Progress Filtering**: Ignores EINPROGRESS status updates
- **Error Propagation**: Stores final error status in wait structure
- **Completion Signaling**: Wakes waiting threads on operation completion
- **Standard Callback**: Common callback for synchronous-style async operations

### 3. Request Cloning Support

**Request Cloning** - Lines 724-739:
```c
struct crypto_async_request *crypto_request_clone(
    struct crypto_async_request *req, size_t total, gfp_t gfp) {
    struct crypto_tfm *tfm = req->tfm;
    struct crypto_async_request *nreq;
    
    nreq = kmemdup(req, total, gfp);
    if (!nreq) {
        req->tfm = tfm->fb;
        return req;
    }
    
    nreq->flags &= ~CRYPTO_TFM_REQ_ON_STACK;
    return nreq;
}
```

**Cloning Features**:
- **Fallback Mechanism**: Uses fallback transform if cloning fails
- **Stack Flag Clearing**: Removes stack allocation flag from cloned requests
- **Memory Duplication**: Complete duplication of request structure
- **GFP Support**: Allows specification of allocation flags

## Advanced Features

### 1. Algorithm State Management

**Algorithm Termination** - Lines 399-405:
```c
void crypto_shoot_alg(struct crypto_alg *alg) {
    down_write(&crypto_alg_sem);
    alg->cra_flags |= CRYPTO_ALG_DYING;
    up_write(&crypto_alg_sem);
}
```

**State Management**:
- **Death Marking**: Marks algorithms as dying to prevent new allocations
- **Synchronized Updates**: Uses write lock for atomic flag updates
- **Cleanup Coordination**: Prevents races during algorithm removal
- **Reference Draining**: Allows existing references to complete

### 2. Context Size Calculation

**Dynamic Context Sizing** - Lines 378-397:
```c
static unsigned int crypto_ctxsize(struct crypto_alg *alg, u32 type, u32 mask) {
    const struct crypto_type *type_obj = alg->cra_type;
    unsigned int len;
    
    len = alg->cra_alignmask & ~(crypto_tfm_ctx_alignment() - 1);
    if (type_obj)
        return len + type_obj->ctxsize(alg, type, mask);
        
    switch (alg->cra_flags & CRYPTO_ALG_TYPE_MASK) {
    default:
        BUG();
        
    case CRYPTO_ALG_TYPE_CIPHER:
        len += crypto_cipher_ctxsize(alg);
        break;
    }
    
    return len;
}
```

**Context Features**:
- **Alignment Handling**: Respects algorithm alignment requirements
- **Type-Specific Sizing**: Uses type-specific context size calculations
- **Legacy Support**: Handles legacy cipher types directly
- **Modern Integration**: Delegates to crypto_type for new algorithm types

### 3. Notification System

**Algorithm Notification** - Lines 323-335:
```c
int crypto_probing_notify(unsigned long val, void *v) {
    int ok;
    
    ok = blocking_notifier_call_chain(&crypto_chain, val, v);
    if (ok == NOTIFY_DONE) {
        request_module("cryptomgr");
        ok = blocking_notifier_call_chain(&crypto_chain, val, v);
    }
    
    return ok;
}
```

**Notification Features**:
- **Event Broadcasting**: Notifies all registered listeners of crypto events
- **Manager Loading**: Automatically loads cryptomgr when needed
- **Retry Mechanism**: Retries notification after loading crypto manager
- **Blocking Semantics**: Uses blocking notifier for synchronous notifications

## Memory Management and Cleanup

### 1. Transform Destruction

**Transform Cleanup** - Lines 670-687:
```c
void crypto_destroy_tfm(void *mem, struct crypto_tfm *tfm) {
    struct crypto_alg *alg;
    
    if (IS_ERR_OR_NULL(mem))
        return;
        
    if (!refcount_dec_and_test(&tfm->refcnt))
        return;
    alg = tfm->__crt_alg;
    
    if (!tfm->exit && alg->cra_exit)
        alg->cra_exit(tfm);
    crypto_exit_ops(tfm);
    crypto_mod_put(alg);
    kfree_sensitive(mem);
}
```

**Cleanup Features**:
- **Reference Counting**: Only destroys when reference count reaches zero
- **Algorithm Cleanup**: Calls algorithm-specific exit functions
- **Type Cleanup**: Calls type-specific exit operations
- **Module References**: Releases module references
- **Secure Clearing**: Uses kfree_sensitive for cryptographic data

### 2. Algorithm Destruction

**Algorithm Cleanup** - Lines 715-722:
```c
void crypto_destroy_alg(struct crypto_alg *alg) {
    if (alg->cra_type && alg->cra_type->destroy)
        alg->cra_type->destroy(alg);
    if (alg->cra_destroy)
        alg->cra_destroy(alg);
}
```

**Destruction Features**:
- **Type-Specific Cleanup**: Calls type-specific destruction functions
- **Algorithm Cleanup**: Calls algorithm-specific destruction functions
- **Hierarchical Cleanup**: Proper ordering of cleanup operations
- **Memory Safety**: Ensures all resources are properly released

## Security Considerations

### 1. FIPS Compliance

**FIPS Mode Support**:
- **Algorithm Filtering**: Blocks non-FIPS algorithms in FIPS mode
- **Test Validation**: Ensures algorithms pass required tests
- **Internal Algorithm Handling**: Manages FIPS-internal algorithms separately
- **Compliance Checking**: Runtime validation of FIPS requirements

### 2. Module Security

**Module Loading Safety**:
- **Reference Counting**: Prevents premature module unloading
- **State Validation**: Checks module state before use
- **Error Handling**: Graceful handling of module loading failures
- **Race Prevention**: Synchronizes module operations

### 3. Memory Security

**Secure Memory Handling**:
- **Sensitive Data Clearing**: Uses kfree_sensitive for cryptographic contexts
- **Reference Tracking**: Prevents use-after-free conditions
- **Atomic Operations**: Thread-safe reference counting
- **Error State Management**: Prevents use of failed algorithms

## Performance Optimizations

### 1. Priority-Based Selection

**Algorithm Prioritization**:
- **Priority Ordering**: Higher priority algorithms preferred
- **Exact Match Priority**: Driver names take precedence over algorithm names
- **Fallback Chains**: Automatic fallback to alternative implementations
- **Cache Efficiency**: Optimized lookup algorithms

### 2. NUMA Awareness

**Node-Local Allocation**:
- **Memory Locality**: Allocates transforms on specified NUMA nodes
- **Performance Optimization**: Reduces memory access latency
- **Scalability**: Better performance on multi-node systems
- **Topology Awareness**: Respects system memory topology

### 3. Lazy Initialization

**Deferred Operations**:
- **Module Loading**: Loads modules only when needed
- **Algorithm Testing**: Tests algorithms on first use
- **Memory Allocation**: Allocates resources when required
- **Reference Management**: Efficient reference counting

The cryptographic core API represents the foundation of Linux's cryptographic infrastructure, providing sophisticated algorithm management, secure memory handling, and high-performance transform allocation while maintaining compatibility with various cryptographic standards and ensuring system security through comprehensive validation and testing mechanisms.