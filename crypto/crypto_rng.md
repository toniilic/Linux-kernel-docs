# Linux Kernel Random Number Generation (rng.c)

## File Purpose and Cryptographic Functionality

The `rng.c` file implements the Random Number Generator (RNG) framework for the Linux kernel's cryptographic subsystem. This file provides the infrastructure for cryptographically secure random number generation, supporting both deterministic and non-deterministic random number generators essential for cryptographic key generation, initialization vectors, nonces, and other security-critical random values.

## Key Data Structures and Algorithms

### RNG Algorithm Framework

- **rng_alg**: Random number generator algorithm definition
  - **generate**: Core random number generation function
  - **seed**: Seeding function for deterministic RNGs
  - **seedsize**: Required/recommended seed size
  - **base**: Underlying crypto_alg structure

### RNG Transform and State Management

- **crypto_rng**: RNG transform instance
  - **base**: Base crypto_tfm structure
  - Algorithm-specific state and configuration
  - Security parameters and entropy state

### Global RNG Management

- **crypto_default_rng**: System-wide default RNG instance
- **crypto_default_rng_refcnt**: Reference count for default RNG
- **crypto_default_rng_lock**: Mutex protecting default RNG operations

## Crypto API Integration and Interfaces

### Random Number Generation Operations

**Primary Generation Interface:**
```c
static inline int crypto_rng_generate(struct crypto_rng *tfm, const u8 *src, unsigned int slen, u8 *dst, unsigned int dlen)
```

**Seeding Operations:**
```c
int crypto_rng_reset(struct crypto_rng *tfm, const u8 *seed, unsigned int slen)
```

**Generation Features:**
- **Configurable Output Length**: Variable-length random number generation
- **Seed Input Support**: Optional seed material for generation
- **Entropy Management**: Proper entropy handling and distribution
- **Algorithm-specific Generation**: Delegates to algorithm implementation

### RNG Seeding Framework

**Advanced Seeding Mechanism:**
```c
int crypto_rng_reset(struct crypto_rng *tfm, const u8 *seed, unsigned int slen)
```

**Seeding Features:**
- **Automatic Seeding**: Uses kernel entropy when no seed provided
- **Custom Seed Support**: Accepts external seed material
- **Entropy Quality**: Integrates with kernel's entropy gathering
- **Secure Memory Handling**: Uses `kfree_sensitive()` for seed material

### Default RNG Management

**System Default RNG:**
```c
int crypto_get_default_rng(void)
void crypto_put_default_rng(void)
```

**Global RNG Features:**
- **Reference Counting**: Safe sharing of default RNG across kernel
- **Lazy Initialization**: Creates default RNG on first access
- **Standard Algorithm**: Uses "stdrng" algorithm for system default
- **Automatic Seeding**: Initializes with kernel entropy

### RNG Algorithm Registration

**Algorithm Registration Framework:**
```c
int crypto_register_rng(struct rng_alg *alg)
void crypto_unregister_rng(struct rng_alg *alg)
```

**Bulk Registration Support:**
```c
int crypto_register_rngs(struct rng_alg *algs, int count)
void crypto_unregister_rngs(struct rng_alg *algs, int count)
```

## Security Considerations and Implementation

### Cryptographic Security Properties

**RNG Security Requirements:**
- **Unpredictability**: Generated values must be cryptographically unpredictable
- **Non-reproducibility**: Successive runs should produce different outputs
- **Uniform Distribution**: Output should be uniformly distributed
- **Entropy Conservation**: Proper entropy management and conservation

### Seed Security Management

**Secure Seeding:**
- **High-Quality Entropy**: Uses kernel's entropy gathering system
- **Seed Protection**: Secure handling of seed material
- **Entropy Estimation**: Proper entropy estimation and management
- **Reseeding Strategy**: Automatic and manual reseeding support

### Default RNG Security

**System RNG Security:**
- **Reference Counting**: Prevents premature RNG destruction
- **Proper Initialization**: Ensures RNG is properly seeded before use
- **Thread Safety**: Mutex protection for default RNG operations
- **Resource Management**: Proper cleanup when RNG no longer needed

### Memory Security

**Secure Memory Operations:**
- **Sensitive Data Cleanup**: Uses `kfree_sensitive()` for seed buffers
- **Buffer Management**: Secure handling of temporary random data
- **State Protection**: Proper protection of internal RNG state

## Dependencies and Subsystem Relationships

### Kernel Entropy Sources

**Entropy Integration:**
- **get_random_bytes_wait()**: Integration with kernel entropy system
- **Hardware RNG**: Support for hardware random number generators
- **Entropy Estimation**: Proper entropy accounting and management
- **Blocking Behavior**: Appropriate blocking when entropy insufficient

### Core Crypto Infrastructure

**Framework Integration:**
- **Crypto API**: Built upon core crypto_tfm infrastructure
- **Type System**: RNG-specific type system integration
- **Algorithm Discovery**: Integration with algorithm registration system

### Hardware Integration

**Hardware RNG Support:**
- **Hardware Abstraction**: Consistent interface for hardware RNGs
- **Fallback Mechanisms**: Software fallback when hardware unavailable
- **Performance Optimization**: Efficient hardware RNG utilization

## Code Flow and Cryptographic Operations

### RNG Algorithm Registration Flow

1. **Validation Phase**:
   - Seed size validation (must not exceed PAGE_SIZE/8)
   - Algorithm structure validation
   - Security parameter checking

2. **Type Configuration**:
   - RNG type flag assignment
   - Type system integration
   - Algorithm capability setup

3. **Registration**:
   - Global algorithm registry integration
   - Algorithm availability announcement

### Default RNG Initialization Flow

1. **First Access**:
   - Default RNG allocation using "stdrng"
   - Algorithm discovery and instantiation
   - Error handling for allocation failures

2. **Seeding Process**:
   - Automatic seeding with kernel entropy
   - Seed size determination from algorithm
   - Error handling for seeding failures

3. **Reference Management**:
   - Reference count initialization
   - Thread-safe access setup
   - Resource sharing preparation

### Random Number Generation Flow

1. **Pre-generation Validation**:
   - RNG state verification
   - Output buffer validation
   - Security parameter checking

2. **Generation Process**:
   - Algorithm-specific generation
   - Entropy management
   - Output length handling

3. **Post-generation**:
   - Output validation
   - Error propagation
   - State updates

## Performance Considerations

### Optimization Strategies

**Efficient Generation:**
- **Hardware Acceleration**: Leverages hardware RNGs when available
- **Batch Generation**: Efficient generation of large random data quantities
- **State Caching**: Optimized internal state management

**Memory Efficiency:**
- **Minimal Allocations**: Reduces memory allocation overhead
- **Buffer Reuse**: Efficient buffer management for generation
- **State Optimization**: Compact internal state representation

### Scalability Features

**High-Performance Random Generation:**
- **Concurrent Access**: Thread-safe random number generation
- **Default RNG Sharing**: Efficient sharing of system RNG
- **Hardware Scaling**: Scales with available hardware RNG capacity

## Advanced Features

### Algorithm-Specific RNG Support

**Diverse RNG Algorithms:**
- **DRBG (Deterministic Random Bit Generator)**: NIST SP 800-90A compliant
- **ChaCha20-based RNGs**: Modern stream cipher-based generation
- **Hardware RNGs**: True random number generator support
- **Hybrid Approaches**: Combining multiple entropy sources

### Entropy Source Integration

**Multiple Entropy Sources:**
- **Kernel Entropy Pool**: Integration with /dev/random system
- **Hardware Entropy**: CPU-based random number instructions
- **Environmental Entropy**: Timing-based entropy collection
- **Cryptographic Entropy**: Algorithm-based entropy stretching

### FIPS Compliance Support

**Regulatory Compliance:**
- **FIPS 140-2**: Support for FIPS-approved RNG algorithms
- **Continuous Testing**: Runtime health monitoring
- **Self-Testing**: Automatic algorithm validation
- **Entropy Validation**: Proper entropy quality assessment

### Default RNG Management

**System-wide RNG Coordination:**
```c
#if defined(CONFIG_CRYPTO_RNG) || defined(CONFIG_CRYPTO_RNG_MODULE)
int crypto_del_default_rng(void)
#endif
```

**Management Features:**
- **Reference Counting**: Safe default RNG lifecycle management
- **Dynamic Replacement**: Ability to change default RNG
- **Resource Cleanup**: Proper cleanup when RNG no longer needed

### Integration with Cryptographic Protocols

**Protocol Support:**
- **Key Generation**: High-quality random keys for symmetric algorithms
- **IV Generation**: Random initialization vectors for block ciphers
- **Nonce Generation**: Unique values for cryptographic protocols
- **Salt Generation**: Random salts for password hashing

### Error Handling and Recovery

**Comprehensive Error Management:**
- **Entropy Starvation**: Graceful handling of insufficient entropy
- **Hardware Failures**: Fallback to software RNGs
- **Algorithm Errors**: Proper error propagation and handling
- **Resource Exhaustion**: Memory and entropy resource management

### Testing and Validation Framework

**RNG Quality Assurance:**
- **Statistical Testing**: Integration with statistical test suites
- **Health Monitoring**: Continuous monitoring of RNG health
- **Performance Testing**: Throughput and latency measurements
- **Security Validation**: Cryptographic strength verification

This RNG implementation provides a secure, efficient, and comprehensive framework for random number generation in the Linux kernel. It supports both deterministic and non-deterministic random number generators while ensuring proper entropy management and security properties essential for cryptographic applications.