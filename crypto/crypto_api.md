# Linux Kernel Crypto API - Main API (api.c)

## File Purpose and Cryptographic Functionality

The `api.c` file is the foundational core of the Linux kernel's cryptographic subsystem, implementing the primary Scatterlist Cryptographic API. This file provides the essential infrastructure for algorithm discovery, registration, allocation, and lifecycle management within the kernel's crypto framework.

## Key Data Structures and Algorithms

### Core Global Data Structures

- **crypto_alg_list**: Global linked list containing all registered cryptographic algorithms
- **crypto_alg_sem**: Read-write semaphore protecting algorithm list operations
- **crypto_chain**: Blocking notifier chain for crypto subsystem events
- **crypto_default_rng**: Default random number generator for the system

### Algorithm Management Structures

- **crypto_larval**: Temporary algorithm placeholder during algorithm loading/testing
  - Contains completion mechanism for asynchronous algorithm loading
  - Handles algorithm testing and validation states
  - Manages adult algorithm references

- **crypto_tfm**: Transform context structure representing algorithm instances
  - Contains algorithm reference and context data
  - Manages reference counting and lifecycle
  - Provides type-specific operations interface

## Crypto API Integration and Interfaces

### Algorithm Discovery and Lookup

The API provides sophisticated algorithm discovery mechanisms:

```c
static struct crypto_alg *__crypto_alg_lookup(const char *name, u32 type, u32 mask)
```

**Algorithm Selection Logic:**
- Exact driver name matches take precedence
- Fuzzy algorithm name matching with priority-based selection
- Supports algorithm inheritance and templating
- FIPS mode compliance checking

### Algorithm Registration Framework

The registration system supports:

- **Single Algorithm Registration**: `crypto_register_alg()`
- **Bulk Registration**: `crypto_register_algs()`
- **Template Registration**: `crypto_register_template()`
- **Instance Registration**: `crypto_register_instance()`

### Transform Allocation and Management

**Multi-level Allocation Strategy:**
1. **Legacy API**: `crypto_alloc_base()` - Generic transform allocation
2. **Type-specific APIs**: `crypto_alloc_tfm()` - Modern approach with frontend types
3. **NUMA-aware allocation**: `crypto_alloc_tfm_node()` - Performance optimization

### Module and Reference Management

**Sophisticated Reference Counting:**
- Algorithm reference counting (`crypto_mod_get/put`)
- Transform reference counting with overflow protection
- Automatic module unloading when algorithms become unused

## Security Considerations and Implementation

### FIPS Mode Support

The API implements comprehensive FIPS 140-2 compliance:

- **Algorithm Validation**: Only FIPS-approved algorithms in FIPS mode
- **Self-testing Framework**: Automatic algorithm validation during registration
- **Runtime Checks**: Continuous validation of algorithm integrity

### Algorithm Testing and Validation

**Larval-based Testing System:**
- Algorithms undergo testing before becoming available
- Failed tests result in algorithm rejection
- Supports both synchronous and asynchronous testing

### Memory Security

**Sensitive Data Protection:**
- Uses `kfree_sensitive()` for cryptographic key material
- Secure memory clearing for temporary buffers
- Protection against memory disclosure attacks

### Access Control

- **Internal Algorithm Flags**: Restricts access to internal-only algorithms
- **Type Masking**: Ensures proper algorithm type usage
- **NOLOAD Flag**: Prevents dynamic module loading when specified

## Dependencies and Subsystem Relationships

### Core Kernel Dependencies

- **Module System**: Integration with kernel module loading/unloading
- **Workqueue**: Asynchronous algorithm testing and cleanup
- **Completion Framework**: Algorithm loading synchronization
- **Notifier Chains**: Event propagation throughout crypto subsystem

### Cryptographic Manager Integration

- **Algorithm Manager (algboss)**: Dynamic algorithm construction
- **Test Manager**: Algorithm validation and testing
- **Template System**: Composite algorithm creation

### Hardware Integration

- **Crypto Engine Framework**: Hardware-accelerated algorithm support
- **Async Framework**: Support for hardware-based asynchronous operations

## Code Flow and Cryptographic Operations

### Algorithm Registration Flow

1. **Validation Phase**:
   - Algorithm structure validation
   - FIPS compliance checking
   - Module signature verification

2. **Testing Phase**:
   - Larval creation for testing
   - Algorithm self-testing
   - Test result validation

3. **Activation Phase**:
   - Algorithm list integration
   - Priority-based positioning
   - Event notification

### Transform Allocation Flow

1. **Discovery Phase**:
   - Algorithm lookup by name/type
   - Priority-based selection
   - Module loading if necessary

2. **Allocation Phase**:
   - Memory allocation (NUMA-aware)
   - Context initialization
   - Algorithm initialization

3. **Activation Phase**:
   - Reference counting setup
   - Type-specific initialization
   - Ready for cryptographic operations

### Algorithm Spawning and Dependencies

**Spawn Management:**
- Tracks algorithm dependencies
- Handles algorithm replacement/updates
- Manages algorithm removal cascades
- Ensures referential integrity

### Error Handling and Recovery

**Comprehensive Error Management:**
- Algorithm loading failures
- Memory allocation failures
- Module loading failures
- Testing and validation failures

## Performance Considerations

### Optimization Strategies

- **Fast Path Algorithm Lookup**: Optimized for common algorithm requests
- **NUMA Awareness**: Memory allocation respecting NUMA topology
- **Lazy Loading**: Dynamic algorithm loading only when needed
- **Caching**: Algorithm instances cached for repeated use

### Scalability Features

- **Read-Write Locking**: Allows concurrent algorithm lookups
- **Reference Counting**: Enables safe concurrent algorithm usage
- **Priority-based Selection**: Ensures optimal algorithm choice

## Advanced Features

### Template System Integration

Support for complex cryptographic constructions:
- Chained algorithms (e.g., HMAC, authenc)
- Mode of operation implementations
- Cryptographic protocol stacks

### Dynamic Algorithm Construction

- Runtime algorithm composition
- Parameter-driven algorithm creation
- Flexible cryptographic primitive combination

### Event Notification System

Comprehensive event propagation for:
- Algorithm registration/unregistration
- Algorithm testing completion
- System state changes

This main API forms the foundation upon which all other cryptographic operations in the Linux kernel are built, providing a robust, secure, and performant framework for cryptographic algorithm management.