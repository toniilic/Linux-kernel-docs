# Linux Kernel Symmetric Key Cipher Operations (skcipher.c)

## File Purpose and Cryptographic Functionality

The `skcipher.c` file implements symmetric key cipher operations for the Linux kernel's cryptographic subsystem. This file provides a comprehensive framework for scatter-gather list processing, block cipher modes of operation, and both synchronous and asynchronous cryptographic processing. It serves as the primary interface for bulk data encryption and decryption operations.

## Key Data Structures and Algorithms

### Symmetric Key Cipher Framework

- **skcipher_alg**: Algorithm definition structure
  - **encrypt/decrypt**: Core operation function pointers
  - **setkey**: Key installation function
  - **min_keysize/max_keysize**: Key size constraints
  - **ivsize**: Initialization vector size
  - **chunksize**: Preferred processing chunk size
  - **walksize**: Scatter-gather walk optimization size

### Request and Context Management

- **skcipher_request**: Request structure for async operations
  - Contains source/destination scatter-gather lists
  - IV (Initialization Vector) management
  - Completion callback for async operations
  - Request flags for operation control

- **skcipher_walk**: Scatter-gather list walker
  - **total**: Total data length to process
  - **nbytes**: Current chunk size
  - **stride**: Algorithm-specific stride length
  - **blocksize/alignmask**: Algorithm constraints

### Transform Types Integration

**Dual Backend Support:**
- **Traditional skcipher**: Classic implementation
- **lskcipher Backend**: Linear skcipher for newer algorithms
- **Transparent Integration**: Automatic backend selection

## Crypto API Integration and Interfaces

### Scatter-Gather List Walking

**Virtual Memory Walking:**
```c
int skcipher_walk_virt(struct skcipher_walk *walk, struct skcipher_request *req, bool atomic)
```

**AEAD Integration Walking:**
```c
int skcipher_walk_aead_encrypt(struct skcipher_walk *walk, struct aead_request *req, bool atomic)
int skcipher_walk_aead_decrypt(struct skcipher_walk *walk, struct aead_request *req, bool atomic)
```

**Walking Features:**
- **Atomic Context Support**: Safe operation in interrupt contexts
- **AEAD Integration**: Seamless integration with authenticated encryption
- **Optimized Chunking**: Algorithm-specific chunk size optimization
- **Memory Management**: Automatic buffer management for processing

### Key Management Operations

**Advanced Key Setting:**
```c
int crypto_skcipher_setkey(struct crypto_skcipher *tfm, const u8 *key, unsigned int keylen)
```

**Key Management Features:**
- **Dual Backend Support**: Handles both skcipher and lskcipher backends
- **Alignment Handling**: Automatic key buffer alignment
- **Need-Key Flag Management**: Tracks key installation state
- **Secure Key Operations**: Uses `kfree_sensitive()` for key buffers

### Encryption and Decryption Operations

**Primary Cryptographic Interface:**
```c
int crypto_skcipher_encrypt(struct skcipher_request *req)
int crypto_skcipher_decrypt(struct skcipher_request *req)
```

**Operation Features:**
- **Backend Abstraction**: Transparent backend selection
- **Key State Validation**: Ensures keys are properly installed
- **Scatter-Gather Support**: Processes complex memory layouts
- **Asynchronous Operations**: Full async operation support

### State Import/Export Framework

**Cryptographic State Management:**
```c
int crypto_skcipher_export(struct skcipher_request *req, void *out)
int crypto_skcipher_import(struct skcipher_request *req, const void *in)
```

**State Management Features:**
- **Backend Compatibility**: Works with both skcipher types
- **IV State Handling**: Manages initialization vector state
- **Algorithm State**: Preserves algorithm-specific state
- **Resumable Operations**: Enables operation suspend/resume

## Security Considerations and Implementation

### Key Security Management

**Secure Key Handling:**
- **Need-Key State**: Prevents operations without proper key installation
- **Secure Memory Cleanup**: Uses `kfree_sensitive()` for sensitive data
- **Key Validation**: Enforces algorithm-specific key size constraints
- **Atomic Key Operations**: Thread-safe key installation

### Memory Security

**Scatter-Gather Security:**
- **Memory Alignment**: Proper handling of unaligned memory access
- **Temporary Buffers**: Secure management of intermediate buffers
- **State Isolation**: Proper isolation between different operations
- **DMA Safety**: Ensures DMA-safe memory handling

### Initialization Vector Management

**IV Security:**
- **IV Validation**: Ensures proper IV handling for different modes
- **IV State Tracking**: Maintains IV state across operations
- **IV Export/Import**: Secure state preservation mechanisms

## Dependencies and Subsystem Relationships

### Core Crypto Infrastructure

**Integration Layers:**
- **Base Crypto API**: Built on fundamental crypto_tfm infrastructure
- **Algorithm Registration**: Integrates with algorithm discovery system
- **Type System**: Provides type-safe symmetric cipher interface

### AEAD Integration

**Authenticated Encryption Support:**
- **AEAD Request Processing**: Processes AEAD requests through skcipher interface
- **Associated Data Handling**: Manages associated data in AEAD operations
- **Authentication Size Management**: Handles authentication tag processing

### Hardware Crypto Engine Integration

**Hardware Acceleration:**
- **Engine Framework**: Integrates with crypto engine for hardware acceleration
- **Async Request Handling**: Supports hardware-based async operations
- **Fallback Mechanisms**: Graceful fallback to software implementations

## Code Flow and Cryptographic Operations

### Algorithm Registration Flow

1. **Algorithm Preparation**:
   - Structure validation and initialization
   - Backend type determination
   - Default function assignment

2. **Registration Process**:
   - Algorithm structure setup
   - Type flag configuration
   - Integration with crypto API

### Request Processing Flow

1. **Request Initialization**:
   - Scatter-gather list setup
   - IV initialization and validation
   - Algorithm constraint checking

2. **Walking Setup**:
   - Walk structure initialization
   - Chunk size optimization
   - Memory layout analysis

3. **Cryptographic Processing**:
   - Backend operation dispatch
   - Scatter-gather list processing
   - State management and IV handling

### Synchronous vs Asynchronous Operations

**Synchronous Allocation:**
```c
struct crypto_sync_skcipher *crypto_alloc_sync_skcipher(const char *alg_name, u32 type, u32 mask)
```

**Sync Operation Features:**
- **Stack-safe Requests**: Limited request size for stack allocation
- **Blocking Operations**: Suitable for contexts that can sleep
- **Error Checking**: Compile-time checks for stack safety

## Performance Considerations

### Optimization Strategies

**Scatter-Gather Optimization:**
- **Walk Size Tuning**: Algorithm-specific walk size optimization
- **Chunk Processing**: Efficient chunk size management
- **Memory Access Patterns**: Optimized memory access for scatter-gather lists

**Backend Selection:**
- **Performance-aware Backend**: Automatic selection of optimal backend
- **Hardware Acceleration**: Transparent hardware acceleration utilization
- **Fallback Performance**: Efficient software fallback implementations

### Memory Management

**Efficient Memory Usage:**
- **Minimal Copies**: Reduces unnecessary memory copying
- **Buffer Reuse**: Efficient buffer management and reuse
- **NUMA Awareness**: Respects NUMA topology for memory allocation

## Advanced Features

### Template-based Simple Modes

**Simple Mode Support:**
```c
struct skcipher_instance *skcipher_alloc_instance_simple(struct crypto_template *tmpl, struct rtattr **tb)
```

**Simple Mode Features:**
- **Template Framework**: Easy implementation of simple cipher modes
- **Default Implementations**: Provides common operation implementations
- **Cipher Integration**: Seamless integration with single-block ciphers
- **Instance Management**: Automatic instance lifecycle management

### Backend Abstraction Layer

**Multi-backend Support:**
- **Legacy Compatibility**: Supports existing skcipher implementations
- **Modern lskcipher**: New linear skcipher backend support
- **Transparent Operation**: Applications unaware of backend differences
- **Performance Optimization**: Backend-specific optimizations

### Request Size Management

**Memory-aware Operations:**
- **Large Request Handling**: Manages large request structures efficiently
- **Stack vs Heap**: Intelligent allocation strategy
- **Request Size Validation**: Prevents oversized stack allocations

### Algorithm Instance Management

**Instance Lifecycle:**
- **Template-based Creation**: Supports template-driven instance creation
- **Dependency Management**: Proper handling of algorithm dependencies
- **Cleanup Automation**: Automatic resource cleanup
- **Error Recovery**: Robust error handling and recovery

### State Preservation

**Advanced State Management:**
- **Cross-request State**: Maintains state across multiple requests
- **Algorithm-specific State**: Preserves algorithm-internal state
- **IV Chain Management**: Proper IV chaining for modes of operation
- **Resume Capabilities**: Enables operation interruption and resumption

### Integration with Modern Crypto Constructions

**Contemporary Protocol Support:**
- **XTS Mode**: Support for full-disk encryption
- **CTR Mode**: Counter mode for streaming encryption
- **AEAD Integration**: Seamless authenticated encryption support
- **ChaCha20-Poly1305**: Modern AEAD cipher support

This symmetric key cipher implementation provides a comprehensive, secure, and high-performance framework for bulk data encryption and decryption in the Linux kernel. It supports both traditional and modern cryptographic algorithms while maintaining compatibility with existing systems and providing pathways for future cryptographic innovations.