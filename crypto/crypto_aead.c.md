# Linux Kernel Authenticated Encryption with Associated Data API (`crypto/aead.c`)

## Overview

The `crypto/aead.c` file implements the Authenticated Encryption with Associated Data (AEAD) API for the Linux kernel's cryptographic subsystem. AEAD algorithms provide both confidentiality and authenticity in a single cryptographic operation, combining encryption with authentication. This module provides the core infrastructure for AEAD algorithm registration, key management, authentication size configuration, and the primary encrypt/decrypt operations. AEAD is essential for secure communications protocols like TLS, IPsec, and other modern cryptographic applications where both data confidentiality and integrity must be guaranteed.

## Core Architecture

### 1. Key Management System

**Aligned Key Setting** - Lines 44-62:
```c
int crypto_aead_setkey(struct crypto_aead *tfm,
                      const u8 *key, unsigned int keylen) {
    unsigned long alignmask = crypto_aead_alignmask(tfm);
    int err;
    
    if ((unsigned long)key & alignmask)
        err = setkey_unaligned(tfm, key, keylen);
    else
        err = crypto_aead_alg(tfm)->setkey(tfm, key, keylen);
        
    if (unlikely(err)) {
        crypto_aead_set_flags(tfm, CRYPTO_TFM_NEED_KEY);
        return err;
    }
    
    crypto_aead_clear_flags(tfm, CRYPTO_TFM_NEED_KEY);
    return 0;
}
```

**Unaligned Key Handling** - Lines 24-42:
```c
static int setkey_unaligned(struct crypto_aead *tfm, const u8 *key,
                           unsigned int keylen) {
    unsigned long alignmask = crypto_aead_alignmask(tfm);
    int ret;
    u8 *buffer, *alignbuffer;
    unsigned long absize;
    
    absize = keylen + alignmask;
    buffer = kmalloc(absize, GFP_ATOMIC);
    if (!buffer)
        return -ENOMEM;
        
    alignbuffer = (u8 *)ALIGN((unsigned long)buffer, alignmask + 1);
    memcpy(alignbuffer, key, keylen);
    ret = crypto_aead_alg(tfm)->setkey(tfm, alignbuffer, keylen);
    kfree_sensitive(buffer);
    return ret;
}
```

**Key Management Features**:
- **Alignment Handling**: Automatically handles unaligned keys through temporary aligned buffers
- **Atomic Allocation**: Uses GFP_ATOMIC for non-sleeping contexts
- **Secure Memory Management**: Uses kfree_sensitive to clear key material from memory
- **Flag Management**: Maintains CRYPTO_TFM_NEED_KEY flag based on key setting success/failure
- **Error Handling**: Proper error propagation and state management
- **Direct Path Optimization**: Fast path for aligned keys, fallback for unaligned

### 2. Authentication Size Configuration

**Authentication Size Setting** - Lines 65-82:
```c
int crypto_aead_setauthsize(struct crypto_aead *tfm, unsigned int authsize) {
    int err;
    
    if ((!authsize && crypto_aead_maxauthsize(tfm)) ||
        authsize > crypto_aead_maxauthsize(tfm))
        return -EINVAL;
        
    if (crypto_aead_alg(tfm)->setauthsize) {
        err = crypto_aead_alg(tfm)->setauthsize(tfm, authsize);
        if (err)
            return err;
    }
    
    tfm->authsize = authsize;
    return 0;
}
```

**Authentication Features**:
- **Size Validation**: Ensures authentication size is within algorithm limits
- **Zero Size Handling**: Validates zero authentication size against maximum allowed
- **Algorithm Callback**: Calls algorithm-specific authentication size handler if available
- **State Management**: Updates transform authentication size after successful validation
- **Range Checking**: Prevents invalid authentication sizes that could compromise security

## Cryptographic Operations

### 1. Encryption Operation

**AEAD Encryption** - Lines 84-93:
```c
int crypto_aead_encrypt(struct aead_request *req) {
    struct crypto_aead *aead = crypto_aead_reqtfm(req);
    
    if (crypto_aead_get_flags(aead) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
        
    return crypto_aead_alg(aead)->encrypt(req);
}
```

**Encryption Features**:
- **Key Validation**: Ensures key is set before allowing encryption operations
- **Direct Algorithm Dispatch**: Routes to algorithm-specific encryption implementation
- **Security Enforcement**: Prevents operations without proper key material
- **Error Propagation**: Returns appropriate error codes for various failure conditions

### 2. Decryption Operation

**AEAD Decryption** - Lines 95-107:
```c
int crypto_aead_decrypt(struct aead_request *req) {
    struct crypto_aead *aead = crypto_aead_reqtfm(req);
    
    if (crypto_aead_get_flags(aead) & CRYPTO_TFM_NEED_KEY)
        return -ENOKEY;
        
    if (req->cryptlen < crypto_aead_authsize(aead))
        return -EINVAL;
        
    return crypto_aead_alg(aead)->decrypt(req);
}
```

**Decryption Features**:
- **Key Validation**: Ensures key is set before allowing decryption operations
- **Length Validation**: Ensures ciphertext is at least as long as authentication tag
- **Security Checks**: Prevents invalid operations that could bypass authentication
- **Algorithm Dispatch**: Routes to algorithm-specific decryption implementation
- **Input Validation**: Comprehensive validation of request parameters

## Transform Lifecycle Management

### 1. Transform Initialization

**Transform Setup** - Lines 117-133:
```c
static int crypto_aead_init_tfm(struct crypto_tfm *tfm) {
    struct crypto_aead *aead = __crypto_aead_cast(tfm);
    struct aead_alg *alg = crypto_aead_alg(aead);
    
    crypto_aead_set_flags(aead, CRYPTO_TFM_NEED_KEY);
    
    aead->authsize = alg->maxauthsize;
    
    if (alg->exit)
        aead->base.exit = crypto_aead_exit_tfm;
        
    if (alg->init)
        return alg->init(aead);
        
    return 0;
}
```

**Initialization Features**:
- **Key State Setup**: Initializes transform to require key setting
- **Default Auth Size**: Sets authentication size to algorithm maximum
- **Exit Handler Setup**: Configures cleanup handler when algorithm provides one
- **Algorithm Initialization**: Calls algorithm-specific initialization if available
- **State Consistency**: Ensures transform is in proper initial state

### 2. Transform Cleanup

**Transform Exit** - Lines 109-115:
```c
static void crypto_aead_exit_tfm(struct crypto_tfm *tfm) {
    struct crypto_aead *aead = __crypto_aead_cast(tfm);
    struct aead_alg *alg = crypto_aead_alg(aead);
    
    alg->exit(aead);
}
```

**Cleanup Features**:
- **Algorithm-Specific Cleanup**: Calls algorithm-specific exit function
- **Resource Deallocation**: Ensures proper cleanup of algorithm resources
- **State Cleanup**: Maintains clean shutdown semantics
- **Memory Management**: Proper cleanup of any allocated resources

## Algorithm Registration Infrastructure

### 1. Algorithm Preparation

**Algorithm Validation** - Lines 213-229:
```c
static int aead_prepare_alg(struct aead_alg *alg) {
    struct crypto_alg *base = &alg->base;
    
    if (max3(alg->maxauthsize, alg->ivsize, alg->chunksize) >
        PAGE_SIZE / 8)
        return -EINVAL;
        
    if (!alg->chunksize)
        alg->chunksize = base->cra_blocksize;
        
    base->cra_type = &crypto_aead_type;
    base->cra_flags &= ~CRYPTO_ALG_TYPE_MASK;
    base->cra_flags |= CRYPTO_ALG_TYPE_AEAD;
    
    return 0;
}
```

**Preparation Features**:
- **Size Limit Enforcement**: Prevents excessive memory usage through size limits
- **Chunk Size Defaulting**: Sets reasonable chunk size defaults when not specified
- **Type Assignment**: Sets proper algorithm type and clears type mask
- **Flag Management**: Ensures proper algorithm flags are set
- **Validation**: Comprehensive validation of algorithm parameters

### 2. Registration Operations

**Single Algorithm Registration** - Lines 231-242:
```c
int crypto_register_aead(struct aead_alg *alg) {
    struct crypto_alg *base = &alg->base;
    int err;
    
    err = aead_prepare_alg(alg);
    if (err)
        return err;
        
    return crypto_register_alg(base);
}
```

**Batch Algorithm Registration** - Lines 250-267:
```c
int crypto_register_aeads(struct aead_alg *algs, int count) {
    int i, ret;
    
    for (i = 0; i < count; i++) {
        ret = crypto_register_aead(&algs[i]);
        if (ret)
            goto err;
    }
    
    return 0;
    
err:
    for (--i; i >= 0; --i)
        crypto_unregister_aead(&algs[i]);
        
    return ret;
}
```

**Registration Features**:
- **Preparation Integration**: Ensures algorithms are properly prepared before registration
- **Batch Support**: Supports registering multiple algorithms atomically
- **Error Recovery**: Proper cleanup on registration failures
- **Atomic Semantics**: All-or-nothing registration for batch operations
- **Rollback Capability**: Unregisters successfully registered algorithms on batch failure

### 3. Algorithm Unregistration

**Single Algorithm Unregistration** - Lines 244-248:
```c
void crypto_unregister_aead(struct aead_alg *alg) {
    crypto_unregister_alg(&alg->base);
}
```

**Batch Algorithm Unregistration** - Lines 270-277:
```c
void crypto_unregister_aeads(struct aead_alg *algs, int count) {
    int i;
    
    for (i = count - 1; i >= 0; --i)
        crypto_unregister_aead(&algs[i]);
}
```

**Unregistration Features**:
- **Simple Interface**: Provides straightforward unregistration
- **Batch Support**: Efficiently unregisters multiple algorithms
- **Reverse Order**: Unregisters in reverse order for proper cleanup
- **Comprehensive Cleanup**: Ensures all algorithms are properly removed

## Instance Management System

### 1. Instance Registration

**Instance Registration** - Lines 279-293:
```c
int aead_register_instance(struct crypto_template *tmpl,
                          struct aead_instance *inst) {
    int err;
    
    if (WARN_ON(!inst->free))
        return -EINVAL;
        
    err = aead_prepare_alg(&inst->alg);
    if (err)
        return err;
        
    return crypto_register_instance(tmpl, aead_crypto_instance(inst));
}
```

**Instance Features**:
- **Free Function Validation**: Ensures instance has proper cleanup function
- **Algorithm Preparation**: Prepares instance algorithm before registration
- **Template Integration**: Registers instance with crypto template system
- **Error Handling**: Proper error propagation from preparation and registration

### 2. Instance Cleanup

**Instance Free** - Lines 168-173:
```c
static void crypto_aead_free_instance(struct crypto_instance *inst) {
    struct aead_instance *aead = aead_instance(inst);
    
    aead->free(aead);
}
```

**Cleanup Features**:
- **Instance-Specific Cleanup**: Calls instance-specific free function
- **Type Safety**: Proper casting from generic instance to AEAD instance
- **Resource Management**: Ensures proper cleanup of instance resources

## Transform Allocation System

### 1. Transform Allocation

**Standard Allocation** - Lines 201-205:
```c
struct crypto_aead *crypto_alloc_aead(const char *alg_name, u32 type, u32 mask) {
    return crypto_alloc_tfm(alg_name, &crypto_aead_type, type, mask);
}
```

**Allocation Features**:
- **Type System Integration**: Uses crypto type system for proper algorithm selection
- **Name-Based Lookup**: Allocates transforms by algorithm name
- **Type and Mask Support**: Supports complex type and mask combinations
- **Error Handling**: Proper error propagation from underlying allocation

### 2. Algorithm Availability

**Algorithm Existence Check** - Lines 207-211:
```c
int crypto_has_aead(const char *alg_name, u32 type, u32 mask) {
    return crypto_type_has_alg(alg_name, &crypto_aead_type, type, mask);
}
```

**Availability Features**:
- **Non-Allocating Check**: Determines algorithm availability without allocation
- **Type-Specific Lookup**: Uses AEAD type system for accurate results
- **Efficient Implementation**: Minimal overhead for availability checking
- **Mask Support**: Supports complex type and mask filtering

## Spawn Management System

### 1. Spawn Operations

**Spawn Grabbing** - Lines 192-199:
```c
int crypto_grab_aead(struct crypto_aead_spawn *spawn,
                    struct crypto_instance *inst,
                    const char *name, u32 type, u32 mask) {
    spawn->base.frontend = &crypto_aead_type;
    return crypto_grab_spawn(&spawn->base, inst, name, type, mask);
}
```

**Spawn Features**:
- **Frontend Assignment**: Sets AEAD type as spawn frontend
- **Instance Integration**: Links spawn to crypto instance
- **Algorithm Discovery**: Uses spawn system for algorithm discovery
- **Type Filtering**: Supports type and mask-based algorithm selection

## Reporting and Debugging Infrastructure

### 1. Algorithm Information Display

**Proc Filesystem Display** - Lines 153-166:
```c
static void crypto_aead_show(struct seq_file *m, struct crypto_alg *alg) {
    struct aead_alg *aead = container_of(alg, struct aead_alg, base);
    
    seq_printf(m, "type         : aead\n");
    seq_printf(m, "async        : %s\n",
               str_yes_no(alg->cra_flags & CRYPTO_ALG_ASYNC));
    seq_printf(m, "blocksize    : %u\n", alg->cra_blocksize);
    seq_printf(m, "ivsize       : %u\n", aead->ivsize);
    seq_printf(m, "maxauthsize  : %u\n", aead->maxauthsize);
    seq_printf(m, "geniv        : <none>\n");
}
```

**Display Features**:
- **Algorithm Type Identification**: Clearly identifies algorithm as AEAD
- **Capability Information**: Shows synchronous/asynchronous capabilities
- **Parameter Display**: Shows all relevant AEAD algorithm parameters
- **Size Information**: Displays block size, IV size, and maximum authentication size
- **Human-Readable Format**: Uses clear formatting for easy reading

### 2. Netlink Reporting

**Netlink Report Generation** - Lines 135-151:
```c
static int __maybe_unused crypto_aead_report(
    struct sk_buff *skb, struct crypto_alg *alg) {
    struct crypto_report_aead raead;
    struct aead_alg *aead = container_of(alg, struct aead_alg, base);
    
    memset(&raead, 0, sizeof(raead));
    
    strscpy(raead.type, "aead", sizeof(raead.type));
    strscpy(raead.geniv, "<none>", sizeof(raead.geniv));
    
    raead.blocksize = alg->cra_blocksize;
    raead.maxauthsize = aead->maxauthsize;
    raead.ivsize = aead->ivsize;
    
    return nla_put(skb, CRYPTOCFGA_REPORT_AEAD, sizeof(raead), &raead);
}
```

**Reporting Features**:
- **Binary Format**: Provides structured binary data for programs
- **Type Identification**: Identifies algorithm type for user-space tools
- **Parameter Export**: Exports key algorithm parameters
- **Netlink Integration**: Uses standard netlink attribute format
- **User-Space Compatibility**: Maintains compatibility with crypto user API

## Crypto Type System Integration

### 1. Type Definition

**AEAD Crypto Type** - Lines 175-190:
```c
static const struct crypto_type crypto_aead_type = {
    .extsize = crypto_alg_extsize,
    .init_tfm = crypto_aead_init_tfm,
    .free = crypto_aead_free_instance,
#ifdef CONFIG_PROC_FS
    .show = crypto_aead_show,
#endif
#if IS_ENABLED(CONFIG_CRYPTO_USER)
    .report = crypto_aead_report,
#endif
    .maskclear = ~CRYPTO_ALG_TYPE_MASK,
    .maskset = CRYPTO_ALG_TYPE_MASK,
    .type = CRYPTO_ALG_TYPE_AEAD,
    .tfmsize = offsetof(struct crypto_aead, base),
    .algsize = offsetof(struct aead_alg, base),
};
```

**Type System Features**:
- **Standard Extension Size**: Uses standard algorithm extension size calculation
- **Transform Lifecycle**: Integrates with transform initialization and cleanup
- **Instance Management**: Provides instance cleanup integration
- **Reporting Integration**: Integrates with proc filesystem and netlink reporting
- **Type Identification**: Proper type identification and masking
- **Size Calculations**: Correct transform and algorithm size calculations

## Security Considerations

### 1. Key Management Security

**Secure Key Handling**:
- **Sensitive Memory Clearing**: Uses kfree_sensitive for key buffers
- **Alignment Security**: Handles unaligned keys without exposing key material
- **Atomic Allocation**: Uses appropriate allocation flags for security contexts  
- **Key State Enforcement**: Prevents operations without proper key material
- **Flag Consistency**: Maintains consistent key requirement flags

### 2. Authentication Security

**Authentication Integrity**:
- **Size Validation**: Enforces proper authentication tag sizes
- **Length Checking**: Validates ciphertext length includes authentication tag
- **Zero Size Handling**: Proper handling of algorithms that don't use authentication
- **Range Enforcement**: Prevents authentication sizes that could compromise security

### 3. Operation Security

**Secure Operations**:
- **Input Validation**: Comprehensive validation of all operation parameters
- **State Verification**: Ensures transforms are in valid states before operations
- **Error Handling**: Secure error handling that doesn't leak information
- **Memory Safety**: Proper memory management throughout all operations

## Performance Optimizations

### 1. Memory Management

**Efficient Memory Usage**:
- **Aligned Buffer Optimization**: Fast path for aligned keys, minimal overhead for unaligned
- **Atomic Allocation**: Appropriate allocation strategies for different contexts
- **Resource Reuse**: Efficient reuse of transform resources
- **Minimal Copying**: Direct algorithm dispatch with minimal overhead

### 2. Algorithm Dispatch

**Efficient Operation Dispatch**:
- **Direct Function Calls**: Minimal overhead for common operations
- **Type-Based Routing**: Fast dispatch based on algorithm type
- **Inline Validation**: Efficient validation with minimal branching
- **Optimized Error Paths**: Fast error handling for common failure cases

### 3. Registration Optimization

**Efficient Registration**:
- **Batch Operations**: Efficient batch registration and unregistration
- **Minimal Validation**: Streamlined validation with necessary checks only
- **Type System Integration**: Efficient integration with crypto type system
- **Memory Layout**: Optimized memory layout for algorithm structures

The AEAD API represents a critical component of the Linux kernel's cryptographic infrastructure, providing secure authenticated encryption capabilities essential for modern cryptographic protocols. Its careful handling of key management, authentication size configuration, and secure operations makes it the foundation for secure communications throughout the kernel, supporting protocols like TLS, IPsec, and other applications requiring both confidentiality and authenticity guarantees.