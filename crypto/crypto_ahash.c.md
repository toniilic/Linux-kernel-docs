# Linux Kernel Asynchronous Hash API (`crypto/ahash.c`)

## Overview

The `crypto/ahash.c` file implements the asynchronous cryptographic hash (ahash) API for the Linux kernel. This subsystem provides a high-level interface for hash operations that supports both asynchronous (ahash) and synchronous (shash) algorithms, with the flexibility to handle data from scatterlists rather than virtually addressed buffers. The ahash API serves as a unified interface that can wrap both native ahash algorithms and shash algorithms, providing asynchronous operation capabilities, scatter-gather I/O support, and sophisticated request chaining mechanisms for complex hash operations.

## Core Architecture

### 1. Hash Walk Infrastructure

**Hash Walk Structure** - Lines 32-43:
```c
struct crypto_hash_walk {
    const char *data;
    
    unsigned int offset;
    unsigned int flags;
    
    struct page *pg;
    unsigned int entrylen;
    
    unsigned int total;
    struct scatterlist *sg;
};
```

**Walk Initialization** - Lines 115-135:
```c
static int crypto_hash_walk_first(struct ahash_request *req,
                                 struct crypto_hash_walk *walk) {
    walk->total = req->nbytes;
    walk->entrylen = 0;
    
    if (!walk->total)
        return 0;
        
    walk->flags = req->base.flags;
    
    if (ahash_request_isvirt(req)) {
        walk->data = req->svirt;
        walk->total = 0;
        return req->nbytes;
    }
    
    walk->sg = req->src;
    
    return hash_walk_new_entry(walk);
}
```

**Walk Features**:
- **Scatterlist Navigation**: Efficiently walks through scatterlist entries
- **Page Management**: Handles data across page boundaries
- **Virtual Buffer Support**: Optimizes for virtually contiguous data
- **Entry Processing**: Manages individual scatterlist entry processing
- **Offset Tracking**: Maintains precise position within pages
- **Total Length Management**: Tracks remaining data to process

### 2. Page Management and Memory Mapping

**Page Navigation** - Lines 86-96:
```c
static int hash_walk_next(struct crypto_hash_walk *walk) {
    unsigned int offset = walk->offset;
    unsigned int nbytes = min(walk->entrylen,
                             ((unsigned int)(PAGE_SIZE)) - offset);
    
    walk->data = kmap_local_page(walk->pg);
    walk->data += offset;
    walk->entrylen -= nbytes;
    return nbytes;
}
```

**Entry Management** - Lines 98-113:
```c
static int hash_walk_new_entry(struct crypto_hash_walk *walk) {
    struct scatterlist *sg;
    
    sg = walk->sg;
    walk->offset = sg->offset;
    walk->pg = nth_page(sg_page(walk->sg), (walk->offset >> PAGE_SHIFT));
    walk->offset = offset_in_page(walk->offset);
    walk->entrylen = sg->length;
    
    if (walk->entrylen > walk->total)
        walk->entrylen = walk->total;
    walk->total -= walk->entrylen;
    
    return hash_walk_next(walk);
}
```

**Memory Management Features**:
- **Local Page Mapping**: Uses kmap_local_page for efficient page access
- **Page Boundary Handling**: Properly handles data spanning multiple pages
- **Offset Calculation**: Computes correct page and offset positions
- **Length Limiting**: Ensures processing doesn't exceed request boundaries
- **Resource Cleanup**: Properly unmaps pages after processing

## SHASH Integration Layer

### 1. SHASH Wrapping Infrastructure

**SHASH Access** - Lines 173-185:
```c
static inline struct crypto_shash *ahash_to_shash(struct crypto_ahash *tfm) {
    return *(struct crypto_shash **)crypto_ahash_ctx(tfm);
}

static inline struct shash_desc *prepare_shash_desc(struct ahash_request *req,
                                                   struct crypto_ahash *tfm) {
    struct shash_desc *desc = ahash_request_ctx(req);
    
    desc->tfm = ahash_to_shash(tfm);
    return desc;
}
```

**SHASH Initialization** - Lines 267-291:
```c
static int crypto_init_ahash_using_shash(struct crypto_tfm *tfm) {
    struct crypto_alg *calg = tfm->__crt_alg;
    struct crypto_ahash *crt = __crypto_ahash_cast(tfm);
    struct crypto_shash **ctx = crypto_tfm_ctx(tfm);
    struct crypto_shash *shash;
    
    if (!crypto_mod_get(calg))
        return -EAGAIN;
        
    shash = crypto_create_tfm(calg, &crypto_shash_type);
    if (IS_ERR(shash)) {
        crypto_mod_put(calg);
        return PTR_ERR(shash);
    }
    
    crt->using_shash = true;
    *ctx = shash;
    tfm->exit = crypto_exit_ahash_using_shash;
    
    crypto_ahash_set_flags(crt, crypto_shash_get_flags(shash) &
                              CRYPTO_TFM_NEED_KEY);
    
    return 0;
}
```

**Integration Features**:
- **Transparent Wrapping**: Wraps shash algorithms in ahash interface
- **Context Management**: Manages shash context within ahash structure
- **Flag Synchronization**: Maintains consistent flags between layers
- **Reference Counting**: Proper module reference management
- **Cleanup Integration**: Sets up proper cleanup handlers

### 2. SHASH Operation Wrappers

**Update Operation** - Lines 187-198:
```c
int shash_ahash_update(struct ahash_request *req, struct shash_desc *desc) {
    struct crypto_hash_walk walk;
    int nbytes;
    
    for (nbytes = crypto_hash_walk_first(req, &walk); nbytes > 0;
         nbytes = crypto_hash_walk_done(&walk, nbytes))
        nbytes = crypto_shash_update(desc, walk.data, nbytes);
        
    return nbytes;
}
```

**Finup Operation** - Lines 200-219:
```c
int shash_ahash_finup(struct ahash_request *req, struct shash_desc *desc) {
    struct crypto_hash_walk walk;
    int nbytes;
    
    nbytes = crypto_hash_walk_first(req, &walk);
    if (!nbytes)
        return crypto_shash_final(desc, req->result);
        
    do {
        nbytes = crypto_hash_walk_last(&walk) ?
                 crypto_shash_finup(desc, walk.data, nbytes,
                                   req->result) :
                 crypto_shash_update(desc, walk.data, nbytes);
        nbytes = crypto_hash_walk_done(&walk, nbytes);
    } while (nbytes > 0);
    
    return nbytes;
}
```

**Digest Operation** - Lines 221-258:
```c
int shash_ahash_digest(struct ahash_request *req, struct shash_desc *desc) {
    unsigned int nbytes = req->nbytes;
    struct scatterlist *sg;
    unsigned int offset;
    struct page *page;
    const u8 *data;
    int err;
    
    data = req->svirt;
    if (!nbytes || ahash_request_isvirt(req))
        return crypto_shash_digest(desc, data, nbytes, req->result);
        
    sg = req->src;
    if (nbytes > sg->length)
        return crypto_shash_init(desc) ?:
               shash_ahash_finup(req, desc);
               
    page = sg_page(sg);
    offset = sg->offset;
    data = lowmem_page_address(page) + offset;
    if (!IS_ENABLED(CONFIG_HIGHMEM))
        return crypto_shash_digest(desc, data, nbytes, req->result);
        
    page = nth_page(page, offset >> PAGE_SHIFT);
    offset = offset_in_page(offset);
    
    if (nbytes > (unsigned int)PAGE_SIZE - offset)
        return crypto_shash_init(desc) ?:
               shash_ahash_finup(req, desc);
               
    data = kmap_local_page(page);
    err = crypto_shash_digest(desc, data + offset, nbytes,
                             req->result);
    kunmap_local(data);
    return err;
}
```

**Operation Features**:
- **Walk-Based Processing**: Uses hash walk infrastructure for scatterlist handling
- **Optimization Paths**: Fast paths for simple cases, fallback for complex ones
- **Memory Efficiency**: Minimizes memory copies through direct access
- **HIGHMEM Support**: Proper handling of high memory pages
- **Error Propagation**: Comprehensive error handling throughout operations

## Key Management System

### 1. Key Setting Infrastructure

**Key Setting** - Lines 306-336:
```c
int crypto_ahash_setkey(struct crypto_ahash *tfm, const u8 *key,
                       unsigned int keylen) {
    if (likely(tfm->using_shash)) {
        struct crypto_shash *shash = ahash_to_shash(tfm);
        int err;
        
        err = crypto_shash_setkey(shash, key, keylen);
        if (unlikely(err)) {
            crypto_ahash_set_flags(tfm,
                                  crypto_shash_get_flags(shash) &
                                  CRYPTO_TFM_NEED_KEY);
            return err;
        }
    } else {
        struct ahash_alg *alg = crypto_ahash_alg(tfm);
        int err;
        
        err = alg->setkey(tfm, key, keylen);
        if (!err && crypto_ahash_need_fallback(tfm))
            err = crypto_ahash_setkey(crypto_ahash_fb(tfm),
                                     key, keylen);
        if (unlikely(err)) {
            ahash_set_needkey(tfm, alg);
            return err;
        }
    }
    crypto_ahash_clear_flags(tfm, CRYPTO_TFM_NEED_KEY);
    return 0;
}
```

**Need Key Management** - Lines 299-304:
```c
static void ahash_set_needkey(struct crypto_ahash *tfm, struct ahash_alg *alg) {
    if (alg->setkey != ahash_nosetkey &&
        !(alg->halg.base.cra_flags & CRYPTO_ALG_OPTIONAL_KEY))
        crypto_ahash_set_flags(tfm, CRYPTO_TFM_NEED_KEY);
}
```

**Key Management Features**:
- **Dual Path Support**: Handles both shash and native ahash key setting
- **Fallback Integration**: Sets keys for fallback transforms when needed
- **Flag Synchronization**: Maintains consistent NEED_KEY flags
- **Error Handling**: Proper error propagation and state management
- **Optional Key Support**: Handles algorithms that don't require keys

## Hash Operation Implementation

### 1. Core Hash Operations

**Initialization** - Lines 381-399:
```c
int crypto_ahash_init(struct ahash_request *req) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    
    if (likely(tfm->using_shash))
        return crypto_shash_init(prepare_shash_desc(req, tfm));
    if (crypto_ahash_get_flags(tfm) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
    if (ahash_req_on_stack(req) && ahash_is_async(tfm))
        return -EAGAIN;
    if (crypto_ahash_block_only(tfm)) {
        u8 *buf = ahash_request_ctx(req);
        
        buf += crypto_ahash_reqsize(tfm) - 1;
        *buf = 0;
    }
    return crypto_ahash_alg(tfm)->init(req);
}
```

**Update Operation** - Lines 453-502:
```c
int crypto_ahash_update(struct ahash_request *req) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    bool nonzero = crypto_ahash_final_nonzero(tfm);
    int bs = crypto_ahash_blocksize(tfm);
    u8 *blenp = ahash_request_ctx(req);
    int blen, err;
    u8 *buf;
    
    if (likely(tfm->using_shash))
        return shash_ahash_update(req, ahash_request_ctx(req));
    if (ahash_req_on_stack(req) && ahash_is_async(tfm))
        return -EAGAIN;
    if (!crypto_ahash_block_only(tfm))
        return ahash_do_req_chain(req, &crypto_ahash_alg(tfm)->update);
        
    blenp += crypto_ahash_reqsize(tfm) - 1;
    blen = *blenp;
    buf = blenp - bs;
    
    if (blen + req->nbytes < bs + nonzero) {
        if (ahash_request_isvirt(req))
            memcpy(buf + blen, req->svirt, req->nbytes);
        else
            memcpy_from_sglist(buf + blen, req->src, 0,
                              req->nbytes);
                              
        *blenp += req->nbytes;
        return 0;
    }
    
    if (blen) {
        memset(req->sg_head, 0, sizeof(req->sg_head[0]));
        sg_set_buf(req->sg_head, buf, blen);
        if (req->src != req->sg_head + 1)
            sg_chain(req->sg_head, 2, req->src);
        req->src = req->sg_head;
        req->nbytes += blen;
    }
    req->nbytes -= nonzero;
    
    ahash_save_req(req, ahash_update_done);
    
    err = ahash_do_req_chain(req, &crypto_ahash_alg(tfm)->update);
    if (err == -EINPROGRESS || err == -EBUSY)
        return err;
        
    return ahash_update_finish(req, err);
}
```

**Operation Features**:
- **Multi-Path Processing**: Handles shash, native ahash, and block-only algorithms
- **Buffering System**: Manages partial blocks for block-only algorithms
- **Scatterlist Chaining**: Chains buffered data with new input data
- **Asynchronous Support**: Proper handling of asynchronous operations
- **Stack Safety**: Prevents stack requests with async algorithms

### 2. Finalization Operations

**Finup Operation** - Lines 534-574:
```c
int crypto_ahash_finup(struct ahash_request *req) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    int bs = crypto_ahash_blocksize(tfm);
    u8 *blenp = ahash_request_ctx(req);
    int blen, err;
    u8 *buf;
    
    if (likely(tfm->using_shash))
        return shash_ahash_finup(req, ahash_request_ctx(req));
    if (ahash_req_on_stack(req) && ahash_is_async(tfm))
        return -EAGAIN;
    if (!crypto_ahash_alg(tfm)->finup)
        return ahash_def_finup(req);
    if (!crypto_ahash_block_only(tfm))
        return ahash_do_req_chain(req, &crypto_ahash_alg(tfm)->finup);
        
    blenp += crypto_ahash_reqsize(tfm) - 1;
    blen = *blenp;
    buf = blenp - bs;
    
    if (blen) {
        memset(req->sg_head, 0, sizeof(req->sg_head[0]));
        sg_set_buf(req->sg_head, buf, blen);
        if (!req->src)
            sg_mark_end(req->sg_head);
        else if (req->src != req->sg_head + 1)
            sg_chain(req->sg_head, 2, req->src);
        req->src = req->sg_head;
        req->nbytes += blen;
    }
    
    ahash_save_req(req, ahash_finup_done);
    
    err = ahash_do_req_chain(req, &crypto_ahash_alg(tfm)->finup);
    if (err == -EINPROGRESS || err == -EBUSY)
        return err;
        
    return ahash_finup_finish(req, err);
}
```

**Digest Operation** - Lines 576-588:
```c
int crypto_ahash_digest(struct ahash_request *req) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    
    if (likely(tfm->using_shash))
        return shash_ahash_digest(req, prepare_shash_desc(req, tfm));
    if (ahash_req_on_stack(req) && ahash_is_async(tfm))
        return -EAGAIN;
    if (crypto_ahash_get_flags(tfm) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
    return ahash_do_req_chain(req, &crypto_ahash_alg(tfm)->digest);
}
```

**Finalization Features**:
- **Default Finup Implementation**: Provides finup via update+final when needed
- **Buffer Management**: Handles buffered data in finalization operations
- **Scatterlist Management**: Proper scatterlist setup for final operations
- **Completion Handling**: Asynchronous completion callback management

## Request Chaining System

### 1. Chain Processing Infrastructure

**Request Chaining** - Lines 338-379:
```c
static int ahash_do_req_chain(struct ahash_request *req,
                             int (*const *op)(struct ahash_request *req)) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    int err;
    
    if (crypto_ahash_req_virt(tfm) || !ahash_request_isvirt(req))
        return (*op)(req);
        
    if (crypto_ahash_statesize(tfm) > HASH_MAX_STATESIZE)
        return -ENOSYS;
        
    {
        u8 state[HASH_MAX_STATESIZE];
        
        if (op == &crypto_ahash_alg(tfm)->digest) {
            ahash_request_set_tfm(req, crypto_ahash_fb(tfm));
            err = crypto_ahash_digest(req);
            goto out_no_state;
        }
        
        err = crypto_ahash_export(req, state);
        ahash_request_set_tfm(req, crypto_ahash_fb(tfm));
        err = err ?: crypto_ahash_import(req, state);
        
        if (op == &crypto_ahash_alg(tfm)->finup) {
            err = err ?: crypto_ahash_finup(req);
            goto out_no_state;
        }
        
        err = err ?:
              crypto_ahash_update(req) ?:
              crypto_ahash_export(req, state);
              
        ahash_request_set_tfm(req, tfm);
        return err ?: crypto_ahash_import(req, state);
        
out_no_state:
        ahash_request_set_tfm(req, tfm);
        return err;
    }
}
```

**Chain Features**:
- **Fallback Integration**: Uses fallback transforms for incompatible requests
- **State Management**: Exports and imports state across transform switches
- **Operation Routing**: Routes operations to appropriate implementations
- **Stack Buffer Optimization**: Uses stack buffers for small state sizes
- **Error Handling**: Comprehensive error handling throughout chain operations

### 2. Asynchronous Completion Handling

**Request State Management** - Lines 401-413:
```c
static void ahash_save_req(struct ahash_request *req, crypto_completion_t cplt) {
    req->saved_complete = req->base.complete;
    req->saved_data = req->base.data;
    req->base.complete = cplt;
    req->base.data = req;
}

static void ahash_restore_req(struct ahash_request *req) {
    req->base.complete = req->saved_complete;
    req->base.data = req->saved_data;
}
```

**Completion Processing** - Lines 65-84:
```c
static inline void ahash_op_done(void *data, int err,
                                int (*finish)(struct ahash_request *, int)) {
    struct ahash_request *areq = data;
    crypto_completion_t compl;
    
    compl = areq->saved_complete;
    data = areq->saved_data;
    if (err == -EINPROGRESS)
        goto out;
        
    areq->base.flags &= ~CRYPTO_TFM_REQ_MAY_SLEEP;
    
    err = finish(areq, err);
    if (err == -EINPROGRESS || err == -EBUSY)
        return;
        
out:
    compl(data, err);
}
```

**Completion Features**:
- **Callback Preservation**: Saves and restores original completion callbacks
- **Context Management**: Maintains proper context through async operations
- **Sleep Flag Management**: Clears sleep flags for completion contexts
- **Error State Handling**: Proper handling of in-progress and busy states

## State Import/Export System

### 1. State Management Infrastructure

**Export Operations** - Lines 637-663:
```c
int crypto_ahash_export(struct ahash_request *req, void *out) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    
    if (likely(tfm->using_shash))
        return crypto_shash_export(ahash_request_ctx(req), out);
    if (crypto_ahash_block_only(tfm)) {
        unsigned int plen = crypto_ahash_blocksize(tfm) + 1;
        unsigned int reqsize = crypto_ahash_reqsize(tfm);
        unsigned int ss = crypto_ahash_statesize(tfm);
        u8 *buf = ahash_request_ctx(req);
        
        memcpy(out + ss - plen, buf + reqsize - plen, plen);
    }
    return crypto_ahash_alg(tfm)->export(req, out);
}
```

**Import Operations** - Lines 678-694:
```c
int crypto_ahash_import(struct ahash_request *req, const void *in) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    
    if (likely(tfm->using_shash))
        return crypto_shash_import(prepare_shash_desc(req, tfm), in);
    if (crypto_ahash_get_flags(tfm) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
    if (crypto_ahash_block_only(tfm)) {
        unsigned int reqsize = crypto_ahash_reqsize(tfm);
        u8 *buf = ahash_request_ctx(req);
        
        buf[reqsize - 1] = 0;
    }
    return crypto_ahash_alg(tfm)->import(req, in);
}
```

**State Features**:
- **Multi-Layer Support**: Handles both shash and native ahash state operations
- **Block-Only Handling**: Special handling for block-only algorithms
- **Buffer Management**: Proper handling of buffered data in state
- **Key Validation**: Ensures keys are set before import operations

### 2. Core State Operations

**Core Export** - Lines 637-645:
```c
int crypto_ahash_export_core(struct ahash_request *req, void *out) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    
    if (likely(tfm->using_shash))
        return crypto_shash_export_core(ahash_request_ctx(req), out);
    return crypto_ahash_alg(tfm)->export_core(req, out);
}
```

**Core Import** - Lines 665-676:
```c
int crypto_ahash_import_core(struct ahash_request *req, const void *in) {
    struct crypto_ahash *tfm = crypto_ahash_reqtfm(req);
    
    if (likely(tfm->using_shash))
        return crypto_shash_import_core(prepare_shash_desc(req, tfm),
                                       in);
    if (crypto_ahash_get_flags(tfm) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
    return crypto_ahash_alg(tfm)->import_core(req, in);
}
```

**Core State Features**:
- **Streamlined Interface**: Simplified state operations without metadata
- **Direct State Access**: Direct access to algorithm internal state
- **Performance Optimization**: Reduced overhead for frequent state operations
- **Consistent Interface**: Uniform interface across different algorithm types

## Transform Lifecycle Management

### 1. Transform Initialization

**Transform Setup** - Lines 710-768:
```c
static int crypto_ahash_init_tfm(struct crypto_tfm *tfm) {
    struct crypto_ahash *hash = __crypto_ahash_cast(tfm);
    struct ahash_alg *alg = crypto_ahash_alg(hash);
    struct crypto_ahash *fb = NULL;
    int err;
    
    crypto_ahash_set_statesize(hash, alg->halg.statesize);
    crypto_ahash_set_reqsize(hash, crypto_tfm_alg_reqsize(tfm));
    
    if (tfm->__crt_alg->cra_type == &crypto_shash_type)
        return crypto_init_ahash_using_shash(tfm);
        
    if (crypto_ahash_need_fallback(hash)) {
        fb = crypto_alloc_ahash(crypto_ahash_alg_name(hash),
                               CRYPTO_ALG_REQ_VIRT,
                               CRYPTO_ALG_ASYNC |
                               CRYPTO_ALG_REQ_VIRT |
                               CRYPTO_AHASH_ALG_NO_EXPORT_CORE);
        if (IS_ERR(fb))
            return PTR_ERR(fb);
            
        tfm->fb = crypto_ahash_tfm(fb);
    }
    
    ahash_set_needkey(hash, alg);
    
    tfm->exit = crypto_ahash_exit_tfm;
    
    if (alg->init_tfm)
        err = alg->init_tfm(hash);
    else if (tfm->__crt_alg->cra_init)
        err = tfm->__crt_alg->cra_init(tfm);
    else
        return 0;
        
    if (err)
        goto out_free_sync_hash;
        
    if (!ahash_is_async(hash) && crypto_ahash_reqsize(hash) >
                                 MAX_SYNC_HASH_REQSIZE)
        goto out_exit_tfm;
        
    BUILD_BUG_ON(HASH_MAX_DESCSIZE > MAX_SYNC_HASH_REQSIZE);
    if (crypto_ahash_reqsize(hash) < HASH_MAX_DESCSIZE)
        crypto_ahash_set_reqsize(hash, HASH_MAX_DESCSIZE);
        
    return 0;
    
out_exit_tfm:
    if (alg->exit_tfm)
        alg->exit_tfm(hash);
    else if (tfm->__crt_alg->cra_exit)
        tfm->__crt_alg->cra_exit(tfm);
    err = -EINVAL;
out_free_sync_hash:
    crypto_free_ahash(fb);
    return err;
}
```

**Initialization Features**:
- **Size Configuration**: Sets up state and request sizes
- **Fallback Allocation**: Allocates fallback transforms when needed
- **Key State Setup**: Initializes key requirement state
- **Request Size Validation**: Ensures appropriate request sizes for sync algorithms
- **Error Recovery**: Proper cleanup on initialization failures

### 2. Transform Cleanup

**Transform Exit** - Lines 696-708:
```c
static void crypto_ahash_exit_tfm(struct crypto_tfm *tfm) {
    struct crypto_ahash *hash = __crypto_ahash_cast(tfm);
    struct ahash_alg *alg = crypto_ahash_alg(hash);
    
    if (alg->exit_tfm)
        alg->exit_tfm(hash);
    else if (tfm->__crt_alg->cra_exit)
        tfm->__crt_alg->cra_exit(tfm);
        
    if (crypto_ahash_need_fallback(hash))
        crypto_free_ahash(crypto_ahash_fb(hash));
}
```

**Cleanup Features**:
- **Algorithm-Specific Cleanup**: Calls algorithm-specific exit functions
- **Fallback Cleanup**: Frees allocated fallback transforms
- **Resource Management**: Ensures all resources are properly released
- **Hierarchical Cleanup**: Follows proper cleanup hierarchy

## Transform Cloning System

### 1. Clone Operations

**Transform Cloning** - Lines 862-930:
```c
struct crypto_ahash *crypto_clone_ahash(struct crypto_ahash *hash) {
    struct hash_alg_common *halg = crypto_hash_alg_common(hash);
    struct crypto_tfm *tfm = crypto_ahash_tfm(hash);
    struct crypto_ahash *fb = NULL;
    struct crypto_ahash *nhash;
    struct ahash_alg *alg;
    int err;
    
    if (!crypto_hash_alg_has_setkey(halg)) {
        tfm = crypto_tfm_get(tfm);
        if (IS_ERR(tfm))
            return ERR_CAST(tfm);
            
        return hash;
    }
    
    nhash = crypto_clone_tfm(&crypto_ahash_type, tfm);
    
    if (IS_ERR(nhash))
        return nhash;
        
    nhash->reqsize = hash->reqsize;
    nhash->statesize = hash->statesize;
    
    if (likely(hash->using_shash)) {
        struct crypto_shash **nctx = crypto_ahash_ctx(nhash);
        struct crypto_shash *shash;
        
        shash = crypto_clone_shash(ahash_to_shash(hash));
        if (IS_ERR(shash)) {
            err = PTR_ERR(shash);
            goto out_free_nhash;
        }
        crypto_ahash_tfm(nhash)->exit = crypto_exit_ahash_using_shash;
        nhash->using_shash = true;
        *nctx = shash;
        return nhash;
    }
    
    if (crypto_ahash_need_fallback(hash)) {
        fb = crypto_clone_ahash(crypto_ahash_fb(hash));
        err = PTR_ERR(fb);
        if (IS_ERR(fb))
            goto out_free_nhash;
            
        crypto_ahash_tfm(nhash)->fb = crypto_ahash_tfm(fb);
    }
    
    err = -ENOSYS;
    alg = crypto_ahash_alg(hash);
    if (!alg->clone_tfm)
        goto out_free_fb;
        
    err = alg->clone_tfm(nhash, hash);
    if (err)
        goto out_free_fb;
        
    crypto_ahash_tfm(nhash)->exit = crypto_ahash_exit_tfm;
    
    return nhash;
    
out_free_fb:
    crypto_free_ahash(fb);
out_free_nhash:
    crypto_free_ahash(nhash);
    return ERR_PTR(err);
}
```

**Cloning Features**:
- **Reference Optimization**: Returns reference for keyless algorithms
- **Deep Cloning**: Creates independent clones for keyed algorithms
- **SHASH Integration**: Properly clones underlying shash transforms
- **Fallback Cloning**: Recursively clones fallback transforms
- **Algorithm Support**: Uses algorithm-specific cloning when available

## Algorithm Registration Infrastructure

### 1. Algorithm Preparation

**Algorithm Validation** - Lines 942-987:
```c
static int ahash_prepare_alg(struct ahash_alg *alg) {
    struct crypto_alg *base = &alg->halg.base;
    int err;
    
    if (alg->halg.statesize == 0)
        return -EINVAL;
        
    if (base->cra_reqsize && base->cra_reqsize < alg->halg.statesize)
        return -EINVAL;
        
    if (!(base->cra_flags & CRYPTO_ALG_ASYNC) &&
        base->cra_reqsize > MAX_SYNC_HASH_REQSIZE)
        return -EINVAL;
        
    err = hash_prepare_alg(&alg->halg);
    if (err)
        return err;
        
    base->cra_type = &crypto_ahash_type;
    base->cra_flags |= CRYPTO_ALG_TYPE_AHASH;
    
    if ((base->cra_flags ^ CRYPTO_ALG_REQ_VIRT) &
        (CRYPTO_ALG_ASYNC | CRYPTO_ALG_REQ_VIRT))
        base->cra_flags |= CRYPTO_ALG_NEED_FALLBACK;
        
    if (!alg->setkey)
        alg->setkey = ahash_nosetkey;
        
    if (base->cra_flags & CRYPTO_AHASH_ALG_BLOCK_ONLY) {
        BUILD_BUG_ON(MAX_ALGAPI_BLOCKSIZE >= 256);
        if (!alg->finup)
            return -EINVAL;
            
        base->cra_reqsize += base->cra_blocksize + 1;
        alg->halg.statesize += base->cra_blocksize + 1;
        alg->export_core = alg->export;
        alg->import_core = alg->import;
    } else if (!alg->export_core || !alg->import_core) {
        alg->export_core = ahash_default_export_core;
        alg->import_core = ahash_default_import_core;
        base->cra_flags |= CRYPTO_AHASH_ALG_NO_EXPORT_CORE;
    }
    
    return 0;
}
```

**Preparation Features**:
- **State Size Validation**: Ensures non-zero state size
- **Request Size Validation**: Validates request size constraints
- **Common Preparation**: Uses shared hash algorithm preparation
- **Fallback Requirements**: Determines fallback needs based on capabilities
- **Block-Only Handling**: Special setup for block-only algorithms
- **Default Handlers**: Sets up default handlers for missing operations

### 2. Registration Operations

**Single Algorithm Registration** - Lines 989-1000:
```c
int crypto_register_ahash(struct ahash_alg *alg) {
    struct crypto_alg *base = &alg->halg.base;
    int err;
    
    err = ahash_prepare_alg(alg);
    if (err)
        return err;
        
    return crypto_register_alg(base);
}
```

**Batch Registration** - Lines 1008-1026:
```c
int crypto_register_ahashes(struct ahash_alg *algs, int count) {
    int i, ret;
    
    for (i = 0; i < count; i++) {
        ret = crypto_register_ahash(&algs[i]);
        if (ret)
            goto err;
    }
    
    return 0;
    
err:
    for (--i; i >= 0; --i)
        crypto_unregister_ahash(&algs[i]);
        
    return ret;
}
```

**Registration Features**:
- **Preparation Integration**: Ensures algorithms are prepared before registration
- **Batch Support**: Supports registering multiple algorithms atomically
- **Error Recovery**: Proper cleanup on registration failures
- **Atomic Semantics**: All-or-nothing registration for batch operations

## Reporting and Debugging Infrastructure

### 1. Algorithm Information Display

**Proc Filesystem Display** - Lines 800-810:
```c
static void crypto_ahash_show(struct seq_file *m, struct crypto_alg *alg) {
    seq_printf(m, "type         : ahash\n");
    seq_printf(m, "async        : %s\n",
               str_yes_no(alg->cra_flags & CRYPTO_ALG_ASYNC));
    seq_printf(m, "blocksize    : %u\n", alg->cra_blocksize);
    seq_printf(m, "digestsize   : %u\n",
               __crypto_hash_alg_common(alg)->digestsize);
}
```

**Netlink Reporting** - Lines 785-798:
```c
static int __maybe_unused crypto_ahash_report(
    struct sk_buff *skb, struct crypto_alg *alg) {
    struct crypto_report_hash rhash;
    
    memset(&rhash, 0, sizeof(rhash));
    
    strscpy(rhash.type, "ahash", sizeof(rhash.type));
    
    rhash.blocksize = alg->cra_blocksize;
    rhash.digestsize = __crypto_hash_alg_common(alg)->digestsize;
    
    return nla_put(skb, CRYPTOCFGA_REPORT_HASH, sizeof(rhash), &rhash);
}
```

**Reporting Features**:
- **Type Identification**: Clearly identifies algorithms as ahash
- **Capability Information**: Shows async capabilities and parameters
- **Size Information**: Displays block and digest sizes
- **User-Space Integration**: Provides information for crypto user API

## Utility Functions

### 1. Hash Digest Helper

**One-Shot Digest** - Lines 1067-1081:
```c
int crypto_hash_digest(struct crypto_ahash *tfm, const u8 *data,
                      unsigned int len, u8 *out) {
    HASH_REQUEST_ON_STACK(req, crypto_ahash_fb(tfm));
    int err;
    
    ahash_request_set_callback(req, 0, NULL, NULL);
    ahash_request_set_virt(req, data, out, len);
    err = crypto_ahash_digest(req);
    
    ahash_request_zero(req);
    
    return err;
}
```

**Helper Features**:
- **Stack Request**: Uses stack-allocated request for efficiency
- **Fallback Usage**: Uses fallback transform for compatibility
- **Security Cleanup**: Zeros request after use
- **Simple Interface**: Provides simple one-shot digest operation

### 2. Request Management

**Request Cleanup** - Lines 1053-1065:
```c
void ahash_request_free(struct ahash_request *req) {
    if (unlikely(!req))
        return;
        
    if (!ahash_req_on_stack(req)) {
        kfree(req);
        return;
    }
    
    ahash_request_zero(req);
}
```

**Management Features**:
- **Null Safety**: Handles null pointer requests safely
- **Stack Detection**: Detects stack vs heap allocated requests
- **Memory Management**: Proper cleanup for heap-allocated requests
- **Security Cleanup**: Zeros stack requests for security

## Performance Optimizations

### 1. Fast Path Optimizations

**Direct Access Optimization**:
- **Virtual Buffer Fast Path**: Direct processing for virtually contiguous data
- **Single Page Optimization**: Fast path for single-page scatterlist entries
- **SHASH Integration**: Minimal overhead for SHASH-wrapped algorithms
- **Stack Request Optimization**: Efficient stack-based request handling

### 2. Memory Management

**Efficient Memory Usage**:
- **Local Page Mapping**: Uses kmap_local_page for better performance
- **Stack Buffer Usage**: Stack buffers for small state sizes
- **Buffer Reuse**: Efficient reuse of request context buffers
- **Minimal Copying**: Direct access to data when possible

### 3. Asynchronous Processing

**Async Optimization**:
- **Callback Chaining**: Efficient callback management for complex operations
- **State Preservation**: Minimal overhead state preservation across async operations
- **Request Chaining**: Efficient chaining of multiple operations
- **Completion Batching**: Batched completion processing where possible

## Security Considerations

### 1. Key Management Security

**Secure Key Handling**:
- **Key State Validation**: Prevents operations without proper keys
- **Flag Consistency**: Maintains consistent key requirement flags
- **Fallback Key Sync**: Ensures fallback transforms have matching keys
- **Error State Security**: Proper error handling prevents information leaks

### 2. Memory Security

**Safe Memory Handling**:
- **Request Zeroing**: Zeros sensitive request data after use
- **Stack Safety**: Prevents stack requests with async algorithms
- **Buffer Management**: Secure handling of internal buffers
- **Page Mapping Safety**: Proper page mapping and unmapping

### 3. State Security

**Secure State Management**:
- **State Export Security**: Validates state before export operations
- **Import Validation**: Ensures proper validation during state import
- **Context Isolation**: Proper isolation between different contexts
- **Error Handling**: Secure error handling throughout state operations

The asynchronous hash API represents a sophisticated and comprehensive interface for hash operations in the Linux kernel. Its careful integration of synchronous and asynchronous algorithms, efficient scatterlist processing, and robust fallback mechanisms make it the foundation for secure and high-performance hash operations throughout the kernel, supporting everything from file integrity checking to cryptographic protocol implementations.