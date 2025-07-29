# Linux Kernel Symmetric Key Cipher API (`crypto/skcipher.c`)

## Overview

The `crypto/skcipher.c` file implements the symmetric key cipher (skcipher) API for the Linux kernel's cryptographic subsystem. This module provides a high-level interface for block and stream ciphers, handling complex operations such as scattergather I/O, cipher walks across page boundaries, key management, and state import/export. It serves as the primary interface for encryption and decryption operations, supporting both synchronous and asynchronous processing models while maintaining compatibility with various cipher modes and underlying algorithm implementations.

## Core Architecture

### 1. Cipher Walk Infrastructure

**Virtual Walk Initialization** - Lines 38-72:
```c
int skcipher_walk_virt(struct skcipher_walk *__restrict walk,
                      struct skcipher_request *__restrict req, bool atomic) {
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    struct skcipher_alg *alg;
    
    might_sleep_if(req->base.flags & CRYPTO_TFM_REQ_MAY_SLEEP);
    
    alg = crypto_skcipher_alg(tfm);
    
    walk->total = req->cryptlen;
    walk->nbytes = 0;
    walk->iv = req->iv;
    walk->oiv = req->iv;
    if (!(req->base.flags & CRYPTO_TFM_REQ_MAY_SLEEP))
        atomic = true;
        
    if (unlikely(!walk->total))
        return 0;
        
    scatterwalk_start(&walk->in, req->src);
    scatterwalk_start(&walk->out, req->dst);
    
    walk->blocksize = crypto_skcipher_blocksize(tfm);
    walk->ivsize = crypto_skcipher_ivsize(tfm);
    walk->alignmask = crypto_skcipher_alignmask(tfm);
    
    if (alg->co.base.cra_type != &crypto_skcipher_type)
        walk->stride = alg->co.chunksize;
    else
        walk->stride = alg->walksize;
        
    return skcipher_walk_first(walk, atomic);
}
```

**Walk Features**:
- **Scattergather Support**: Handles complex memory layouts across multiple pages
- **Atomic Operation Support**: Supports both atomic and sleeping contexts
- **IV Management**: Manages initialization vectors and original IV preservation
- **Alignment Handling**: Respects cipher alignment requirements
- **Stride Optimization**: Uses algorithm-specific stride for efficient processing
- **Sleep Context Detection**: Automatically determines atomic requirements

### 2. AEAD Integration

**AEAD Encrypt Walk** - Lines 100-108:
```c
int skcipher_walk_aead_encrypt(struct skcipher_walk *__restrict walk,
                              struct aead_request *__restrict req,
                              bool atomic) {
    walk->total = req->cryptlen;
    
    return skcipher_walk_aead_common(walk, req, atomic);
}
```

**AEAD Decrypt Walk** - Lines 110-120:
```c
int skcipher_walk_aead_decrypt(struct skcipher_walk *__restrict walk,
                              struct aead_request *__restrict req,
                              bool atomic) {
    struct crypto_aead *tfm = crypto_aead_reqtfm(req);
    
    walk->total = req->cryptlen - crypto_aead_authsize(tfm);
    
    return skcipher_walk_aead_common(walk, req, atomic);
}
```

**AEAD Integration Features**:
- **Associated Data Handling**: Skips associated data during cipher walks
- **Authentication Size Adjustment**: Excludes authentication tag from ciphertext
- **Unified Walk Interface**: Provides consistent walk interface for AEAD operations
- **Position Management**: Handles scattergather position offsets correctly

## Key Management System

### 1. Key Setting Infrastructure

**Aligned Key Setting** - Lines 149-184:
```c
int crypto_skcipher_setkey(struct crypto_skcipher *tfm, const u8 *key,
                          unsigned int keylen) {
    struct skcipher_alg *cipher = crypto_skcipher_alg(tfm);
    unsigned long alignmask = crypto_skcipher_alignmask(tfm);
    int err;
    
    if (cipher->co.base.cra_type != &crypto_skcipher_type) {
        struct crypto_lskcipher **ctx = crypto_skcipher_ctx(tfm);
        
        crypto_lskcipher_clear_flags(*ctx, CRYPTO_TFM_REQ_MASK);
        crypto_lskcipher_set_flags(*ctx,
                                  crypto_skcipher_get_flags(tfm) &
                                  CRYPTO_TFM_REQ_MASK);
        err = crypto_lskcipher_setkey(*ctx, key, keylen);
        goto out;
    }
    
    if (keylen < cipher->min_keysize || keylen > cipher->max_keysize)
        return -EINVAL;
        
    if ((unsigned long)key & alignmask)
        err = skcipher_setkey_unaligned(tfm, key, keylen);
    else
        err = cipher->setkey(tfm, key, keylen);
        
out:
    if (unlikely(err)) {
        skcipher_set_needkey(tfm);
        return err;
    }
    
    crypto_skcipher_clear_flags(tfm, CRYPTO_TFM_NEED_KEY);
    return 0;
}
```

**Key Management Features**:
- **Key Size Validation**: Enforces minimum and maximum key size constraints
- **Alignment Handling**: Automatically handles unaligned keys through temporary buffers
- **LSKCIPHER Support**: Delegates to linear skcipher for appropriate algorithm types
- **Flag Synchronization**: Maintains consistent flags between transform and context
- **Need Key State**: Manages CRYPTO_TFM_NEED_KEY flag based on key setting success
- **Error Recovery**: Sets need key state on key setting failures

### 2. Unaligned Key Handling

**Unaligned Key Processing** - Lines 128-147:
```c
static int skcipher_setkey_unaligned(struct crypto_skcipher *tfm,
                                    const u8 *key, unsigned int keylen) {
    unsigned long alignmask = crypto_skcipher_alignmask(tfm);
    struct skcipher_alg *cipher = crypto_skcipher_alg(tfm);
    u8 *buffer, *alignbuffer;
    unsigned long absize;
    int ret;
    
    absize = keylen + alignmask;
    buffer = kmalloc(absize, GFP_ATOMIC);
    if (!buffer)
        return -ENOMEM;
        
    alignbuffer = (u8 *)ALIGN((unsigned long)buffer, alignmask + 1);
    memcpy(alignbuffer, key, keylen);
    ret = cipher->setkey(tfm, alignbuffer, keylen);
    kfree_sensitive(buffer);
    return ret;
}
```

**Alignment Features**:
- **Dynamic Buffer Allocation**: Allocates aligned buffer for unaligned keys
- **Atomic Allocation**: Uses GFP_ATOMIC for non-sleeping contexts
- **Memory Security**: Uses kfree_sensitive to clear key material
- **Alignment Calculation**: Computes proper alignment based on algorithm requirements
- **Temporary Buffer Management**: Ensures proper cleanup after key setting

### 3. Need Key State Management

**Need Key Setting** - Lines 122-126:
```c
static void skcipher_set_needkey(struct crypto_skcipher *tfm) {
    if (crypto_skcipher_max_keysize(tfm) != 0)
        crypto_skcipher_set_flags(tfm, CRYPTO_TFM_NEED_KEY);
}
```

**State Features**:
- **Conditional Flag Setting**: Only sets need key flag for key-requiring algorithms
- **Zero Key Size Handling**: Handles algorithms that don't require keys
- **State Consistency**: Maintains consistent key requirement state
- **Security Enforcement**: Prevents operations without proper key material

## Encryption and Decryption Operations

### 1. Primary Cryptographic Operations

**Encryption Operation** - Lines 186-197:
```c
int crypto_skcipher_encrypt(struct skcipher_request *req) {
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    struct skcipher_alg *alg = crypto_skcipher_alg(tfm);
    
    if (crypto_skcipher_get_flags(tfm) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
    if (alg->co.base.cra_type != &crypto_skcipher_type)
        return crypto_lskcipher_encrypt_sg(req);
    return alg->encrypt(req);
}
```

**Decryption Operation** - Lines 199-210:
```c
int crypto_skcipher_decrypt(struct skcipher_request *req) {
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    struct skcipher_alg *alg = crypto_skcipher_alg(tfm);
    
    if (crypto_skcipher_get_flags(tfm) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
    if (alg->co.base.cra_type != &crypto_skcipher_type)
        return crypto_lskcipher_decrypt_sg(req);
    return alg->decrypt(req);
}
```

**Operation Features**:
- **Key Validation**: Enforces key requirement before operations
- **Algorithm Type Dispatch**: Routes to appropriate implementation based on algorithm type
- **LSKCIPHER Integration**: Seamlessly handles linear skcipher algorithms
- **Error Propagation**: Returns appropriate error codes for various failure conditions
- **Unified Interface**: Provides consistent API regardless of underlying implementation

## State Import/Export System

### 1. State Management Infrastructure

**State Export** - Lines 248-257:
```c
int crypto_skcipher_export(struct skcipher_request *req, void *out) {
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    struct skcipher_alg *alg = crypto_skcipher_alg(tfm);
    
    if (alg->co.base.cra_type != &crypto_skcipher_type)
        return crypto_lskcipher_export(req, out);
    return alg->export(req, out);
}
```

**State Import** - Lines 259-268:
```c
int crypto_skcipher_import(struct skcipher_request *req, const void *in) {
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    struct skcipher_alg *alg = crypto_skcipher_alg(tfm);
    
    if (alg->co.base.cra_type != &crypto_skcipher_type)
        return crypto_lskcipher_import(req, in);
    return alg->import(req, in);
}
```

**State Management Features**:
- **Algorithm Type Routing**: Delegates to appropriate implementation
- **Binary State Handling**: Supports opaque state import/export
- **Streaming Support**: Enables resumable cryptographic operations
- **Context Preservation**: Maintains algorithm-specific internal state

### 2. LSKCIPHER State Handling

**LSKCIPHER Export** - Lines 212-223:
```c
static int crypto_lskcipher_export(struct skcipher_request *req, void *out) {
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    u8 *ivs = skcipher_request_ctx(req);
    
    ivs = PTR_ALIGN(ivs, crypto_skcipher_alignmask(tfm) + 1);
    
    memcpy(out, ivs + crypto_skcipher_ivsize(tfm),
           crypto_skcipher_statesize(tfm));
           
    return 0;
}
```

**LSKCIPHER Import** - Lines 225-236:
```c
static int crypto_lskcipher_import(struct skcipher_request *req, const void *in) {
    struct crypto_skcipher *tfm = crypto_skcipher_reqtfm(req);
    u8 *ivs = skcipher_request_ctx(req);
    
    ivs = PTR_ALIGN(ivs, crypto_skcipher_alignmask(tfm) + 1);
    
    memcpy(ivs + crypto_skcipher_ivsize(tfm), in,
           crypto_skcipher_statesize(tfm));
           
    return 0;
}
```

**LSKCIPHER State Features**:
- **Alignment Handling**: Properly aligns state data in request context
- **Memory Layout Management**: Handles IV and state data layout
- **Direct Memory Operations**: Uses efficient memory copy operations
- **Size Validation**: Respects algorithm-specific state sizes

## Transform Lifecycle Management

### 1. Transform Initialization

**Transform Initialization** - Lines 278-304:
```c
static int crypto_skcipher_init_tfm(struct crypto_tfm *tfm) {
    struct crypto_skcipher *skcipher = __crypto_skcipher_cast(tfm);
    struct skcipher_alg *alg = crypto_skcipher_alg(skcipher);
    
    skcipher_set_needkey(skcipher);
    
    if (tfm->__crt_alg->cra_type != &crypto_skcipher_type) {
        unsigned am = crypto_skcipher_alignmask(skcipher);
        unsigned reqsize;
        
        reqsize = am & ~(crypto_tfm_ctx_alignment() - 1);
        reqsize += crypto_skcipher_ivsize(skcipher);
        reqsize += crypto_skcipher_statesize(skcipher);
        crypto_skcipher_set_reqsize(skcipher, reqsize);
        
        return crypto_init_lskcipher_ops_sg(tfm);
    }
    
    if (alg->exit)
        skcipher->base.exit = crypto_skcipher_exit_tfm;
        
    if (alg->init)
        return alg->init(skcipher);
        
    return 0;
}
```

**Initialization Features**:
- **Need Key State Setup**: Initializes key requirement state
- **Request Size Calculation**: Computes appropriate request context size
- **LSKCIPHER Integration**: Sets up linear skcipher operations when needed
- **Exit Handler Setup**: Configures cleanup handlers when provided
- **Algorithm-Specific Init**: Calls algorithm-specific initialization if available

### 2. Transform Cleanup

**Transform Exit** - Lines 270-276:
```c
static void crypto_skcipher_exit_tfm(struct crypto_tfm *tfm) {
    struct crypto_skcipher *skcipher = __crypto_skcipher_cast(tfm);
    struct skcipher_alg *alg = crypto_skcipher_alg(skcipher);
    
    alg->exit(skcipher);
}
```

**Cleanup Features**:
- **Algorithm-Specific Cleanup**: Calls algorithm-specific exit function
- **Resource Deallocation**: Ensures proper cleanup of algorithm resources
- **State Cleanup**: Maintains clean shutdown semantics

## Algorithm Registration Infrastructure

### 1. Algorithm Preparation

**Algorithm Preparation** - Lines 441-466:
```c
static int skcipher_prepare_alg(struct skcipher_alg *alg) {
    struct crypto_alg *base = &alg->base;
    int err;
    
    err = skcipher_prepare_alg_common(&alg->co);
    if (err)
        return err;
        
    if (alg->walksize > PAGE_SIZE / 8)
        return -EINVAL;
        
    if (!alg->walksize)
        alg->walksize = alg->chunksize;
        
    if (!alg->statesize) {
        alg->import = skcipher_noimport;
        alg->export = skcipher_noexport;
    } else if (!(alg->import && alg->export))
        return -EINVAL;
        
    base->cra_type = &crypto_skcipher_type;
    base->cra_flags |= CRYPTO_ALG_TYPE_SKCIPHER;
    
    return 0;
}
```

**Preparation Features**:
- **Common Validation**: Validates common algorithm parameters
- **Walk Size Management**: Sets appropriate walk size defaults
- **State Handler Setup**: Configures import/export handlers based on state requirements
- **Type Assignment**: Sets proper algorithm type and flags
- **Consistency Validation**: Ensures import/export functions are properly paired

### 2. Common Algorithm Validation

**Common Preparation** - Lines 424-439:
```c
int skcipher_prepare_alg_common(struct skcipher_alg_common *alg) {
    struct crypto_alg *base = &alg->base;
    
    if (alg->ivsize > PAGE_SIZE / 8 || alg->chunksize > PAGE_SIZE / 8 ||
        alg->statesize > PAGE_SIZE / 2 ||
        (alg->ivsize + alg->statesize) > PAGE_SIZE / 2)
        return -EINVAL;
        
    if (!alg->chunksize)
        alg->chunksize = base->cra_blocksize;
        
    base->cra_flags &= ~CRYPTO_ALG_TYPE_MASK;
    
    return 0;
}
```

**Validation Features**:
- **Size Limit Enforcement**: Prevents excessive memory usage through size limits
- **Chunk Size Defaulting**: Sets reasonable chunk size defaults
- **Flag Cleanup**: Clears type mask for proper type assignment
- **Memory Usage Control**: Ensures reasonable memory requirements

### 3. Registration Operations

**Single Algorithm Registration** - Lines 468-479:
```c
int crypto_register_skcipher(struct skcipher_alg *alg) {
    struct crypto_alg *base = &alg->base;
    int err;
    
    err = skcipher_prepare_alg(alg);
    if (err)
        return err;
        
    return crypto_register_alg(base);
}
```

**Batch Registration** - Lines 487-505:
```c
int crypto_register_skciphers(struct skcipher_alg *algs, int count) {
    int i, ret;
    
    for (i = 0; i < count; i++) {
        ret = crypto_register_skcipher(&algs[i]);
        if (ret)
            goto err;
    }
    
    return 0;
    
err:
    for (--i; i >= 0; --i)
        crypto_unregister_skcipher(&algs[i]);
        
    return ret;
}
```

**Registration Features**:
- **Preparation Integration**: Ensures algorithms are properly prepared before registration
- **Batch Operations**: Supports registering multiple algorithms atomically
- **Error Recovery**: Properly unregisters partially registered algorithms on failure
- **Atomic Semantics**: Either all algorithms register successfully or none do

## Transform Allocation System

### 1. General Transform Allocation

**Standard Allocation** - Lines 386-391:
```c
struct crypto_skcipher *crypto_alloc_skcipher(const char *alg_name,
                                             u32 type, u32 mask) {
    return crypto_alloc_tfm(alg_name, &crypto_skcipher_type, type, mask);
}
```

**Synchronous Allocation** - Lines 393-416:
```c
struct crypto_sync_skcipher *crypto_alloc_sync_skcipher(
                const char *alg_name, u32 type, u32 mask) {
    struct crypto_skcipher *tfm;
    
    /* Only sync algorithms allowed. */
    mask |= CRYPTO_ALG_ASYNC | CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE;
    type &= ~(CRYPTO_ALG_ASYNC | CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE);
    
    tfm = crypto_alloc_tfm(alg_name, &crypto_skcipher_type, type, mask);
    
    /*
     * Make sure we do not allocate something that might get used with
     * an on-stack request: check the request size.
     */
    if (!IS_ERR(tfm) && WARN_ON(crypto_skcipher_reqsize(tfm) >
                                MAX_SYNC_SKCIPHER_REQSIZE)) {
        crypto_free_skcipher(tfm);
        return ERR_PTR(-EINVAL);
    }
    
    return (struct crypto_sync_skcipher *)tfm;
}
```

**Allocation Features**:
- **Type System Integration**: Uses crypto type system for proper algorithm selection
- **Synchronous Restrictions**: Enforces synchronous-only algorithms for sync interface
- **Request Size Validation**: Ensures request sizes are suitable for stack allocation
- **Safety Checks**: Prevents allocation of inappropriate algorithms for sync use
- **Error Handling**: Proper cleanup on validation failures

### 2. Algorithm Availability Checking

**Algorithm Existence Check** - Lines 418-422:
```c
int crypto_has_skcipher(const char *alg_name, u32 type, u32 mask) {
    return crypto_type_has_alg(alg_name, &crypto_skcipher_type, type, mask);
}
```

**Availability Features**:
- **Non-Allocating Check**: Determines algorithm availability without allocation
- **Type-Specific Lookup**: Uses skcipher type system for accurate results
- **Mask Support**: Supports complex type and mask combinations
- **Efficient Implementation**: Minimal overhead for availability checking

## Simple Cipher Mode Support

### 1. Simple Block Cipher Integration

**Simple Key Setting** - Lines 532-541:
```c
static int skcipher_setkey_simple(struct crypto_skcipher *tfm, const u8 *key,
                                 unsigned int keylen) {
    struct crypto_cipher *cipher = skcipher_cipher_simple(tfm);
    
    crypto_cipher_clear_flags(cipher, CRYPTO_TFM_REQ_MASK);
    crypto_cipher_set_flags(cipher, crypto_skcipher_get_flags(tfm) &
                           CRYPTO_TFM_REQ_MASK);
    return crypto_cipher_setkey(cipher, key, keylen);
}
```

**Simple Transform Management** - Lines 543-563:
```c
static int skcipher_init_tfm_simple(struct crypto_skcipher *tfm) {
    struct skcipher_instance *inst = skcipher_alg_instance(tfm);
    struct crypto_cipher_spawn *spawn = skcipher_instance_ctx(inst);
    struct skcipher_ctx_simple *ctx = crypto_skcipher_ctx(tfm);
    struct crypto_cipher *cipher;
    
    cipher = crypto_spawn_cipher(spawn);
    if (IS_ERR(cipher))
        return PTR_ERR(cipher);
        
    ctx->cipher = cipher;
    return 0;
}

static void skcipher_exit_tfm_simple(struct crypto_skcipher *tfm) {
    struct skcipher_ctx_simple *ctx = crypto_skcipher_ctx(tfm);
    
    crypto_free_cipher(ctx->cipher);
}
```

**Simple Mode Features**:
- **Flag Synchronization**: Maintains consistent flags between skcipher and cipher
- **Spawn Integration**: Uses spawn system for underlying cipher allocation
- **Context Management**: Manages simple context structure with cipher pointer
- **Resource Cleanup**: Proper cleanup of underlying cipher resources

### 2. Simple Instance Allocation

**Instance Allocation Helper** - Lines 587-637:
```c
struct skcipher_instance *skcipher_alloc_instance_simple(
    struct crypto_template *tmpl, struct rtattr **tb) {
    u32 mask;
    struct skcipher_instance *inst;
    struct crypto_cipher_spawn *spawn;
    struct crypto_alg *cipher_alg;
    int err;
    
    err = crypto_check_attr_type(tb, CRYPTO_ALG_TYPE_SKCIPHER, &mask);
    if (err)
        return ERR_PTR(err);
        
    inst = kzalloc(sizeof(*inst) + sizeof(*spawn), GFP_KERNEL);
    if (!inst)
        return ERR_PTR(-ENOMEM);
    spawn = skcipher_instance_ctx(inst);
    
    err = crypto_grab_cipher(spawn, skcipher_crypto_instance(inst),
                            crypto_attr_alg_name(tb[1]), 0, mask);
    if (err)
        goto err_free_inst;
    cipher_alg = crypto_spawn_cipher_alg(spawn);
    
    err = crypto_inst_setname(skcipher_crypto_instance(inst), tmpl->name,
                             cipher_alg);
    if (err)
        goto err_free_inst;
        
    inst->free = skcipher_free_instance_simple;
    
    /* Default algorithm properties, can be overridden */
    inst->alg.base.cra_blocksize = cipher_alg->cra_blocksize;
    inst->alg.base.cra_alignmask = cipher_alg->cra_alignmask;
    inst->alg.base.cra_priority = cipher_alg->cra_priority;
    inst->alg.min_keysize = cipher_alg->cra_cipher.cia_min_keysize;
    inst->alg.max_keysize = cipher_alg->cra_cipher.cia_max_keysize;
    inst->alg.ivsize = cipher_alg->cra_blocksize;
    
    /* Use skcipher_ctx_simple by default, can be overridden */
    inst->alg.base.cra_ctxsize = sizeof(struct skcipher_ctx_simple);
    inst->alg.setkey = skcipher_setkey_simple;
    inst->alg.init = skcipher_init_tfm_simple;
    inst->alg.exit = skcipher_exit_tfm_simple;
    
    return inst;
    
err_free_inst:
    skcipher_free_instance_simple(inst);
    return ERR_PTR(err);
}
```

**Instance Allocation Features**:
- **Attribute Validation**: Validates template parameters and type compatibility
- **Memory Management**: Allocates instance and spawn structures together
- **Cipher Spawning**: Creates spawn for underlying cipher algorithm
- **Name Generation**: Generates appropriate instance names
- **Property Inheritance**: Inherits properties from underlying cipher
- **Default Method Setup**: Provides complete default implementation
- **Error Recovery**: Proper cleanup on allocation failures
- **Customization Support**: Allows overriding of default properties and methods

## Reporting and Debugging Infrastructure

### 1. Algorithm Information Display

**Proc Filesystem Display** - Lines 322-338:
```c
static void crypto_skcipher_show(struct seq_file *m, struct crypto_alg *alg) {
    struct skcipher_alg *skcipher = __crypto_skcipher_alg(alg);
    
    seq_printf(m, "type         : skcipher\n");
    seq_printf(m, "async        : %s\n",
               str_yes_no(alg->cra_flags & CRYPTO_ALG_ASYNC));
    seq_printf(m, "blocksize    : %u\n", alg->cra_blocksize);
    seq_printf(m, "min keysize  : %u\n", skcipher->min_keysize);
    seq_printf(m, "max keysize  : %u\n", skcipher->max_keysize);
    seq_printf(m, "ivsize       : %u\n", skcipher->ivsize);
    seq_printf(m, "chunksize    : %u\n", skcipher->chunksize);
    seq_printf(m, "walksize     : %u\n", skcipher->walksize);
    seq_printf(m, "statesize    : %u\n", skcipher->statesize);
}
```

**Display Features**:
- **Algorithm Type Identification**: Clearly identifies algorithm as skcipher
- **Capability Information**: Shows synchronous/asynchronous capabilities
- **Parameter Display**: Shows all relevant algorithm parameters
- **Size Information**: Displays various size parameters for debugging
- **Human-Readable Format**: Uses clear formatting for easy reading

### 2. Netlink Reporting

**Netlink Report Generation** - Lines 340-358:
```c
static int crypto_skcipher_report(struct sk_buff *skb, struct crypto_alg *alg) {
    struct skcipher_alg *skcipher = __crypto_skcipher_alg(alg);
    struct crypto_report_blkcipher rblkcipher;
    
    memset(&rblkcipher, 0, sizeof(rblkcipher));
    
    strscpy(rblkcipher.type, "skcipher", sizeof(rblkcipher.type));
    strscpy(rblkcipher.geniv, "<none>", sizeof(rblkcipher.geniv));
    
    rblkcipher.blocksize = alg->cra_blocksize;
    rblkcipher.min_keysize = skcipher->min_keysize;
    rblkcipher.max_keysize = skcipher->max_keysize;
    rblkcipher.ivsize = skcipher->ivsize;
    
    return nla_put(skb, CRYPTOCFGA_REPORT_BLKCIPHER,
                   sizeof(rblkcipher), &rblkcipher);
}
```

**Reporting Features**:
- **Binary Format**: Provides structured binary data for programs
- **Type Identification**: Identifies algorithm type for user-space tools
- **Parameter Export**: Exports key algorithm parameters
- **Netlink Integration**: Uses standard netlink attribute format
- **User-Space Compatibility**: Maintains compatibility with crypto user API

## Security Considerations

### 1. Key Material Handling

**Secure Key Processing**:
- **Sensitive Memory Clearing**: Uses kfree_sensitive for key buffers
- **Alignment Security**: Handles unaligned keys without exposing key material
- **Atomic Allocation**: Uses appropriate allocation flags for security contexts
- **Key Validation**: Enforces key size constraints before processing

### 2. State Management Security

**Secure State Handling**:
- **Need Key Enforcement**: Prevents operations without proper key material
- **State Validation**: Validates state sizes and alignment requirements
- **Context Protection**: Proper alignment and size management for contexts
- **Memory Layout Security**: Ensures secure memory layout for IV and state data

### 3. Algorithm Validation

**Registration Security**:
- **Size Limit Enforcement**: Prevents resource exhaustion through size limits
- **Parameter Validation**: Comprehensive validation of algorithm parameters
- **Type System Integration**: Uses secure type system for algorithm selection
- **Consistency Checks**: Ensures proper pairing of import/export functions

## Performance Optimizations

### 1. Memory Management

**Optimized Memory Usage**:
- **Aligned Allocations**: Respects algorithm alignment requirements
- **Context Size Optimization**: Calculates minimal required context sizes
- **Stack-Friendly Design**: Supports stack allocation for synchronous operations
- **Memory Layout Optimization**: Efficient layout of IV, state, and context data

### 2. Algorithm Dispatch

**Efficient Operation Dispatch**:
- **Type-Based Routing**: Fast dispatch based on algorithm type
- **Direct Function Calls**: Minimal overhead for common operations
- **Inline Helpers**: Uses inline functions for performance-critical paths
- **Branch Prediction**: Optimizes common code paths

### 3. Walk Optimization

**Efficient Data Processing**:
- **Stride-Based Processing**: Uses algorithm-specific stride for optimal performance
- **Scattergather Optimization**: Efficient handling of complex memory layouts
- **Page Boundary Handling**: Optimized processing across page boundaries
- **Atomic Context Support**: Efficient operation in both atomic and sleeping contexts

The symmetric key cipher API represents a sophisticated and comprehensive interface for cryptographic operations in the Linux kernel. Its careful handling of memory management, security requirements, and performance optimization makes it the foundation for secure and efficient symmetric cryptography throughout the kernel, supporting everything from disk encryption to network security protocols.