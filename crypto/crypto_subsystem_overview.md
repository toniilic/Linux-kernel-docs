# Linux Kernel Crypto Subsystem - Comprehensive Overview

## Introduction

The Linux kernel's cryptographic subsystem provides a unified, high-performance, and secure framework for cryptographic operations throughout the kernel. This documentation covers the core files that form the foundation of this subsystem, providing comprehensive analysis of their purpose, functionality, and integration.

## Core Crypto Framework Files

### 1. Core API Infrastructure

#### [crypto_api.md](crypto_api.md) - Main Crypto API (api.c)
**Purpose**: Foundational core implementing the Scatterlist Cryptographic API
- **Key Functions**: Algorithm discovery, registration, allocation, and lifecycle management
- **Security Features**: FIPS mode support, algorithm testing framework, memory security
- **Integration**: Module system, workqueue framework, completion mechanisms

#### [crypto_algapi.md](crypto_algapi.md) - Algorithm API (algapi.c)  
**Purpose**: Low-level Algorithm API for complex cryptographic constructions
- **Key Functions**: Template management, instance handling, dependency resolution
- **Security Features**: FIPS compliance, algorithm testing, secure cleanup
- **Integration**: Template system, algorithm dependency management, spawn lifecycle

### 2. Symmetric Cryptographic Operations

#### [crypto_cipher.md](crypto_cipher.md) - Single-Block Cipher (cipher.c)
**Purpose**: Single-block cipher operations with memory alignment handling
- **Key Functions**: Block encryption/decryption, key management, cipher cloning
- **Security Features**: Secure key handling, alignment protection, atomic operations
- **Integration**: Building block for block cipher modes, hardware integration

#### [crypto_skcipher.md](crypto_skcipher.md) - Symmetric Key Cipher (skcipher.c)
**Purpose**: Symmetric key cipher framework for bulk data operations
- **Key Functions**: Scatter-gather processing, async/sync operations, state management
- **Security Features**: IV management, key validation, secure memory handling
- **Integration**: AEAD support, hardware engines, multiple backend support

#### [crypto_shash.md](crypto_shash.md) - Synchronous Hash (shash.c)
**Purpose**: Synchronous hash operations for immediate completion contexts
- **Key Functions**: Hash computation, HMAC support, state export/import
- **Security Features**: Keyed hash protection, context isolation, side-channel resistance
- **Integration**: Block-only algorithm support, template framework, cloning

### 3. Advanced Cryptographic Primitives

#### [crypto_aead.md](crypto_aead.md) - Authenticated Encryption (aead.c)
**Purpose**: Authenticated Encryption with Associated Data for secure communications
- **Key Functions**: Combined encryption/authentication, tag management, protocol support
- **Security Features**: Authentication tag validation, associated data protection
- **Integration**: Modern protocol support (TLS 1.3, IPsec), hardware acceleration

#### [crypto_akcipher.md](crypto_akcipher.md) - Asymmetric Cipher (akcipher.c)
**Purpose**: Public key cryptography for digital signatures and key exchange
- **Key Functions**: Public/private key operations, digital signatures, key management
- **Security Features**: Key validation, computational security, side-channel protection
- **Integration**: PKI infrastructure, certificate processing, authentication systems

#### [crypto_kpp.md](crypto_kpp.md) - Key-agreement Protocols (kpp.c)
**Purpose**: Key agreement primitives for secure key establishment
- **Key Functions**: Diffie-Hellman, ECDH, shared secret computation
- **Security Features**: Forward secrecy, key validation, protocol security
- **Integration**: Modern protocols (TLS 1.3, WireGuard), post-quantum preparation

#### [crypto_rng.md](crypto_rng.md) - Random Number Generation (rng.c)
**Purpose**: Cryptographically secure random number generation
- **Key Functions**: DRBG implementation, entropy management, default RNG
- **Security Features**: Entropy quality, seed protection, FIPS compliance
- **Integration**: Kernel entropy sources, hardware RNG, protocol support

### 4. Hardware Acceleration Framework

#### [crypto_crypto_engine.md](crypto_crypto_engine.md) - Crypto Engine (crypto_engine.c)
**Purpose**: Hardware crypto engine framework for accelerated operations
- **Key Functions**: Request queuing, hardware coordination, async processing
- **Security Features**: Secure hardware integration, resource protection
- **Integration**: All crypto types, power management, performance optimization

## System Architecture and Integration

### Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kernel Subsystems                        │
│           (Networking, Storage, Security, etc.)             │
├─────────────────────────────────────────────────────────────┤
│                  High-Level Crypto APIs                     │
│          (skcipher, aead, akcipher, kpp, rng, shash)        │
├─────────────────────────────────────────────────────────────┤
│                  Core Crypto Framework                      │
│              (api.c, algapi.c, cipher.c)                   │
├─────────────────────────────────────────────────────────────┤
│                Hardware Acceleration Layer                  │
│                   (crypto_engine.c)                        │
├─────────────────────────────────────────────────────────────┤
│              Algorithm Implementations                      │
│        (AES, SHA, RSA, ECC, ChaCha20, Poly1305, etc.)      │
└─────────────────────────────────────────────────────────────┘
```

### Security Architecture

#### Multi-Level Security Framework
1. **Algorithm Level**: Individual algorithm security properties
2. **Framework Level**: Type safety, memory protection, state isolation
3. **System Level**: FIPS compliance, hardware security, entropy management
4. **Protocol Level**: Integration with secure communication protocols

#### Memory Security
- **Sensitive Data Protection**: `kfree_sensitive()` usage throughout
- **Alignment Handling**: Secure memory alignment for hardware compatibility
- **Buffer Management**: DMA-safe memory handling
- **State Isolation**: Proper context isolation between operations

### Performance Architecture

#### Optimization Strategies
1. **Hardware Acceleration**: Seamless integration with crypto hardware
2. **Async Operations**: Full asynchronous operation support
3. **Memory Efficiency**: Optimized memory access patterns
4. **Algorithm Selection**: Priority-based optimal algorithm selection

#### Scalability Features
- **NUMA Awareness**: Memory allocation respecting NUMA topology
- **Parallel Processing**: Support for concurrent crypto operations
- **Load Balancing**: Intelligent distribution across hardware engines
- **Resource Management**: Efficient resource allocation and cleanup

## Cryptographic Algorithm Support

### Symmetric Cryptography
- **Block Ciphers**: AES, ChaCha20, SM4, Camellia
- **Hash Functions**: SHA-256/512, SHA-3, BLAKE2, SM3
- **Authentication**: HMAC, Poly1305, GHASH
- **Modes**: GCM, CCM, CTR, CBC, XTS

### Asymmetric Cryptography
- **Digital Signatures**: RSA, ECDSA, EdDSA, ECRDSA
- **Key Exchange**: ECDH, X25519, X448, traditional DH
- **Encryption**: RSA-OAEP, RSA-PKCS1

### Modern Constructions
- **AEAD**: ChaCha20-Poly1305, AES-GCM, AES-CCM
- **Key Derivation**: HKDF, PBKDF2, scrypt
- **Random Generation**: DRBG (CTR, Hash, HMAC)

## Integration with Kernel Subsystems

### Network Stack Integration
- **IPsec**: ESP/AH protocol crypto operations
- **TLS**: Kernel TLS implementation support
- **WireGuard**: Modern VPN crypto primitives
- **MAC protocols**: Various network authentication

### Storage Subsystem Integration
- **dm-crypt**: Full disk encryption
- **fscrypt**: File system level encryption
- **LUKS**: Linux Unified Key Setup
- **Integrity checking**: File and block integrity

### Security Framework Integration
- **Trusted Platform Module (TPM)**: Hardware security integration
- **Hardware Security Modules (HSM)**: Secure key storage
- **Kernel keyring**: Cryptographic key management
- **IMA/EVM**: Integrity measurement and verification

## Hardware Crypto Engine Support

### Supported Hardware
- **x86 Crypto Extensions**: AES-NI, SHA extensions, AVX
- **ARM Crypto Extensions**: ARMv8 crypto instructions
- **Hardware Accelerators**: Dedicated crypto hardware
- **Smart Cards**: Crypto card integration

### Engine Framework Benefits
- **Unified Interface**: Consistent API across different hardware
- **Fallback Support**: Automatic software fallback
- **Performance Optimization**: Hardware-specific optimizations
- **Resource Management**: Efficient hardware resource utilization

## Security Compliance and Standards

### FIPS 140-2 Compliance
- **Approved Algorithms**: Only FIPS-approved algorithms in FIPS mode
- **Self-Testing**: Comprehensive algorithm validation
- **Entropy Requirements**: Proper entropy handling
- **Key Management**: Secure cryptographic key lifecycle

### Industry Standards
- **NIST Guidelines**: Compliance with NIST cryptographic standards
- **RFC Implementations**: Standard protocol implementations
- **Common Criteria**: Security evaluation criteria support
- **Industry Protocols**: TLS, IPsec, SSH, etc.

## Future Developments

### Post-Quantum Cryptography
- **Algorithm Preparation**: Framework ready for post-quantum algorithms
- **Key Exchange**: Post-quantum key encapsulation mechanisms
- **Digital Signatures**: Post-quantum signature schemes
- **Migration Strategy**: Smooth transition from classical to post-quantum

### Emerging Technologies
- **Hardware Security**: Enhanced hardware security integration
- **Cloud Security**: Cloud-specific crypto optimizations
- **IoT Security**: Lightweight crypto for IoT devices
- **AI/ML Security**: Cryptographic protection for AI/ML workloads

## Conclusion

The Linux kernel crypto subsystem provides a comprehensive, secure, and high-performance foundation for cryptographic operations. The documented core files work together to create a robust framework that supports everything from basic symmetric encryption to advanced post-quantum cryptographic protocols, while maintaining strict security properties and providing excellent performance across diverse hardware platforms.

This subsystem continues to evolve to meet emerging security challenges while maintaining backward compatibility and providing the foundation for secure computing in the Linux ecosystem.