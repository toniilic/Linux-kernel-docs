# net/socket.c - Linux Socket Layer Implementation

## Overview

This file implements the core socket layer in the Linux networking subsystem. Originally written by various developers including Alan Cox, Mike Shaver, and others, it provides the fundamental BSD socket paradigm interface that applications use to access network protocols. This implementation serves as the bridge between user space applications and the underlying network protocol stacks.

## Historical Development

### Key Contributors and Milestones
- **Early Unix Origins**: Based on BSD socket design from UC Berkeley
- **1990s Linux Integration**: Adapted for Linux by Linus Torvalds and network developers
- **Alan Cox Era**: Significant networking stack development and stability improvements
- **Modern Enhancements**: Performance optimizations, security hardening, and feature additions

### Evolution Timeline
- **1991-1994**: Basic socket implementation for Linux
- **1995-2000**: Performance improvements and protocol family expansion
- **2000-2010**: Security enhancements, namespace support, and scalability improvements
- **2010-Present**: Modern features like io_uring integration, eBPF hooks, and container support

## Core Concepts

### Socket Abstraction
The socket layer provides a unified interface that abstracts different network protocols:

#### Socket Types
- **SOCK_STREAM**: Reliable, sequenced, connection-based byte streams (TCP)
- **SOCK_DGRAM**: Unreliable, connectionless datagrams (UDP)
- **SOCK_SEQPACKET**: Reliable, sequenced, connection-based packets
- **SOCK_RAW**: Raw network protocol access
- **SOCK_RDM**: Reliable delivered messages
- **SOCK_PACKET**: Obsolete packet interface

#### Protocol Families
The system supports numerous protocol families:
- **PF_INET**: IPv4 Internet protocols
- **PF_INET6**: IPv6 Internet protocols  
- **PF_UNIX**: Unix domain sockets
- **PF_NETLINK**: Netlink communication
- **PF_PACKET**: Low-level packet interface
- **PF_BLUETOOTH**: Bluetooth protocols
- **PF_CAN**: Controller Area Network
- **And many more specialized families**

### Socket States
- **SS_UNCONNECTED**: Socket created but not connected
- **SS_CONNECTING**: Connection in progress
- **SS_CONNECTED**: Socket connected
- **SS_DISCONNECTING**: Disconnection in progress

## Key Data Structures

### `struct socket` - Core Socket Structure
```c
struct socket {
    socket_state        state;          /* Socket state */
    short              type;           /* Socket type */
    unsigned long      flags;          /* Socket flags */
    struct file        *file;          /* Associated file structure */
    struct sock        *sk;            /* Network protocol sock */
    const struct proto_ops *ops;       /* Protocol operations */
    struct socket_wq   wq;             /* Wait queue */
};
```

**Key Fields**:
- **state**: Current connection state
- **type**: Socket type (STREAM, DGRAM, etc.)
- **ops**: Protocol-specific operations
- **sk**: Lower-level protocol socket structure
- **file**: VFS file structure for file operations

### `struct socket_alloc` - Socket Allocation Container
```c
struct socket_alloc {
    struct socket socket;       /* Socket structure */
    struct inode vfs_inode;    /* VFS inode */
};
```

### `struct proto_ops` - Protocol Operations
```c
struct proto_ops {
    int family;                 /* Protocol family */
    struct module *owner;       /* Owner module */
    
    int (*release)(struct socket *sock);
    int (*bind)(struct socket *sock, struct sockaddr *addr, int addr_len);
    int (*connect)(struct socket *sock, struct sockaddr *addr, int addr_len, int flags);
    int (*socketpair)(struct socket *sock1, struct socket *sock2);
    int (*accept)(struct socket *sock, struct socket *newsock, struct proto_accept_arg *arg);
    int (*getname)(struct socket *sock, struct sockaddr *addr, int peer);
    __poll_t (*poll)(struct file *file, struct socket *sock, struct poll_table_struct *wait);
    int (*ioctl)(struct socket *sock, unsigned int cmd, unsigned long arg);
    int (*listen)(struct socket *sock, int len);
    int (*shutdown)(struct socket *sock, int flags);
    int (*setsockopt)(struct socket *sock, int level, int optname, sockptr_t optval, unsigned int optlen);
    int (*getsockopt)(struct socket *sock, int level, int optname, char __user *optval, int __user *optlen);
    int (*sendmsg)(struct socket *sock, struct msghdr *m, size_t total_len);
    int (*recvmsg)(struct socket *sock, struct msghdr *m, size_t total_len, int flags);
    int (*mmap)(struct file *file, struct socket *sock, struct vm_area_struct *vma);
};
```

### File Operations Integration
```c
static const struct file_operations socket_file_ops = {
    .owner =        THIS_MODULE,
    .read_iter =    sock_read_iter,
    .write_iter =   sock_write_iter,
    .poll =         sock_poll,
    .unlocked_ioctl = sock_ioctl,
    .compat_ioctl = compat_sock_ioctl,
    .uring_cmd =    io_uring_cmd_sock,
    .mmap =         sock_mmap,
    .release =      sock_close,
    .fasync =       sock_fasync,
    .splice_write = splice_to_socket,
    .splice_read =  sock_splice_read,
    .splice_eof =   sock_splice_eof,
    .show_fdinfo =  sock_show_fdinfo,
};
```

## Core Functions

### Socket Creation and Management

#### `__sock_create()` - Core Socket Creation
```c
int __sock_create(struct net *net, int family, int type, int protocol,
                 struct socket **res, int kern)
```

**Purpose**: Creates a new socket with specified parameters

**Creation Process**:
1. **Parameter Validation**: Check family, type, and protocol validity
2. **Socket Allocation**: Allocate socket structure and VFS inode
3. **Protocol Family Lookup**: Find appropriate protocol family handler
4. **Module Management**: Handle loadable protocol modules
5. **Protocol Creation**: Call protocol-specific create function
6. **Security Checks**: Apply security policies and auditing
7. **Result Return**: Return created socket to caller

**Key Features**:
- **Module Loading**: Automatic module loading for missing protocols
- **Reference Counting**: Proper module reference management
- **Error Handling**: Comprehensive error cleanup
- **Security Integration**: LSM (Linux Security Module) hooks

#### `sock_alloc()` - Socket Structure Allocation
```c
struct socket *sock_alloc(void)
```

**Allocation Process**:
1. **Inode Creation**: Create pseudo-filesystem inode
2. **Socket Initialization**: Initialize socket structure
3. **VFS Integration**: Set up inode operations
4. **Wait Queue Setup**: Initialize wait queues

#### `sock_release()` - Socket Cleanup
```c
void sock_release(struct socket *sock)
```

**Cleanup Process**:
1. **Protocol Release**: Call protocol-specific release
2. **Module Reference**: Drop module references
3. **File Association**: Clean up file associations
4. **Resource Cleanup**: Free socket resources

### Address Handling

#### `move_addr_to_kernel()` - User to Kernel Address Copy
```c
int move_addr_to_kernel(void __user *uaddr, int ulen, struct sockaddr_storage *kaddr)
```

**Address Processing**:
1. **Length Validation**: Check address length bounds
2. **User Copy**: Safely copy from user space
3. **Audit Integration**: Record address for security auditing
4. **Error Handling**: Handle copy failures and invalid addresses

#### `move_addr_to_user()` - Kernel to User Address Copy
```c
static int move_addr_to_user(struct sockaddr_storage *kaddr, int klen,
                            void __user *uaddr, int __user *ulen)
```

**User Copy Process**:
1. **Buffer Length Check**: Verify user buffer size
2. **Truncation Handling**: Handle buffer overflow situations
3. **Audit Checks**: Security audit validation
4. **Copy to User**: Safe kernel-to-user copying

### System Call Implementations

#### `SYSCALL_DEFINE3(socket, ...)`
```c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
```

**Socket Creation Syscall**:
1. **Parameter Processing**: Extract and validate syscall parameters
2. **Socket Creation**: Create socket using `__sys_socket`
3. **File Descriptor**: Allocate and bind file descriptor
4. **Return Value**: Return file descriptor or error

#### `SYSCALL_DEFINE3(bind, ...)`
```c
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
```

**Address Binding Process**:
1. **File Descriptor Lookup**: Verify socket file descriptor
2. **Address Copy**: Copy address from user space
3. **Security Checks**: Apply binding security policies
4. **Protocol Binding**: Call protocol-specific bind operation

#### `SYSCALL_DEFINE2(listen, ...)`
```c
SYSCALL_DEFINE2(listen, int, fd, int, backlog)
```

**Listen Setup**:
1. **Socket Validation**: Verify socket is appropriate for listening
2. **Backlog Processing**: Apply system limits to backlog size
3. **Security Checks**: Validate listen permissions
4. **Protocol Listen**: Enable protocol-specific listening

#### `SYSCALL_DEFINE4(accept4, ...)`
```c
SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
               int __user *, upeer_addrlen, int, flags)
```

**Connection Acceptance**:
1. **Socket Allocation**: Create new socket for accepted connection
2. **File Creation**: Set up file structure for new socket
3. **Protocol Accept**: Call protocol-specific accept function
4. **Address Return**: Copy peer address to user space
5. **File Descriptor**: Install new file descriptor

#### `SYSCALL_DEFINE3(connect, ...)`
```c
SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr, int, addrlen)
```

**Connection Establishment**:
1. **Address Processing**: Copy and validate target address
2. **Security Checks**: Apply connection security policies
3. **Protocol Connect**: Initiate protocol-specific connection
4. **Blocking Behavior**: Handle blocking vs. non-blocking modes

### Message Handling

#### `____sys_sendmsg()` - Core Message Sending
```c
static int ____sys_sendmsg(struct socket *sock, struct msghdr *msg_sys,
                          unsigned int flags, struct used_address *used_address,
                          unsigned int allowed_msghdr_flags)
```

**Message Sending Process**:
1. **Control Message Processing**: Handle ancillary data
2. **Flag Processing**: Process message flags and file flags
3. **Security Optimization**: Skip redundant security checks for repeated addresses
4. **Protocol Transmission**: Call protocol-specific send function
5. **Address Caching**: Cache successful addresses for efficiency

#### Message Structure Handling
```c
int __copy_msghdr(struct msghdr *kmsg, struct user_msghdr *msg,
                 struct sockaddr __user **save_addr)
```

**Message Header Processing**:
- **Address Handling**: Process destination/source addresses
- **Control Data**: Handle ancillary data and control messages
- **I/O Vector**: Set up scatter-gather I/O vectors
- **Flag Processing**: Handle message transmission flags

### Socket Options and Control

#### `do_sock_setsockopt()` - Socket Option Setting
```c
int do_sock_setsockopt(struct socket *sock, bool compat, int level,
                      int optname, sockptr_t optval, unsigned int optlen)
```

**Option Processing**:
1. **Security Validation**: Check option setting permissions
2. **Level Routing**: Route to socket vs. protocol level
3. **eBPF Integration**: Apply eBPF program modifications
4. **Protocol Delegation**: Call protocol-specific option handlers

#### `do_sock_getsockopt()` - Socket Option Retrieval
```c
int do_sock_getsockopt(struct socket *sock, bool compat, int level,
                      int optname, sockptr_t optval, sockptr_t optlen)
```

**Option Retrieval**:
1. **Security Checks**: Validate option access permissions
2. **Buffer Management**: Handle option value buffers
3. **Protocol Queries**: Query protocol-specific options
4. **eBPF Processing**: Apply eBPF program filters

### Special Operations

#### `SYSCALL_DEFINE4(socketpair, ...)`
```c
SYSCALL_DEFINE4(socketpair, int, family, int, type, int, protocol,
               int __user *, usockvec)
```

**Socketpair Creation**:
1. **Dual Socket Creation**: Create two connected sockets
2. **Cross-Connection**: Establish bidirectional connection
3. **File Descriptor Allocation**: Allocate two file descriptors
4. **Security Validation**: Apply security policies to pair
5. **Return Descriptors**: Return both descriptors to user

#### `__sys_shutdown()` - Socket Shutdown
```c
int __sys_shutdown(int fd, int how)
```

**Shutdown Processing**:
- **Direction Control**: Shutdown read, write, or both directions
- **Protocol Notification**: Inform protocol of shutdown
- **Security Checks**: Validate shutdown permissions

## File System Integration

### Socket Filesystem (sockfs)
```c
static const struct super_operations sockfs_ops = {
    .alloc_inode = sock_alloc_inode,
    .free_inode = sock_free_inode,
    .statfs = simple_statfs,
};
```

**Pseudo-Filesystem Features**:
- **Inode Management**: Custom inode allocation for sockets
- **Dentry Operations**: Special dentry naming for socket identification
- **Extended Attributes**: Support for socket metadata
- **Security Labels**: Integration with security frameworks

#### Socket Inode Operations
```c
static const struct inode_operations sockfs_inode_ops = {
    .listxattr = sockfs_listxattr,
    .setattr = sockfs_setattr,
};
```

**Extended Attributes**:
- **Protocol Names**: Expose protocol information
- **Security Labels**: Support LSM security labels
- **Permission Management**: Handle socket ownership changes

### File Descriptor Integration

#### `sock_alloc_file()` - File Structure Binding
```c
struct file *sock_alloc_file(struct socket *sock, int flags, const char *dname)
```

**File Integration**:
1. **Pseudo-File Creation**: Create file structure for socket
2. **Operations Binding**: Link socket operations to file operations
3. **Flag Processing**: Handle file status flags
4. **Name Assignment**: Set appropriate file name
5. **Notification Setup**: Configure filesystem notifications

#### File Operation Implementations
- **Read/Write**: Implement read/write through socket operations
- **Poll**: Provide polling interface for I/O readiness
- **Ioctl**: Handle socket-specific control operations
- **Mmap**: Support memory mapping where applicable

## Performance Optimizations

### Fast Path Optimizations
- **Address Caching**: Cache recently used addresses in sendmsg
- **Control Message Optimization**: Efficient handling of ancillary data
- **File Operation Inlining**: Direct socket operation calls
- **Reference Counting**: Optimized reference management

### I/O Performance
- **Zero-Copy Operations**: Support for splice and mmap operations
- **Vectored I/O**: Efficient scatter-gather operations
- **Asynchronous I/O**: Integration with io_uring for async operations
- **Polling Optimization**: Efficient readiness notification

### Memory Management
- **Slab Caching**: Custom slab cache for socket inodes
- **Buffer Management**: Efficient buffer allocation and management
- **Reference Optimization**: Minimal reference counting overhead

## Security Framework Integration

### Linux Security Modules (LSM)
The socket layer provides extensive LSM hooks:

#### Socket Lifecycle Hooks
- **socket_create**: Control socket creation
- **socket_bind**: Control address binding
- **socket_connect**: Control connection establishment
- **socket_listen**: Control listening setup
- **socket_accept**: Control connection acceptance

#### Data Transfer Hooks
- **socket_sendmsg**: Control message transmission
- **socket_recvmsg**: Control message reception
- **socket_getsockopt**: Control option retrieval
- **socket_setsockopt**: Control option setting

### Audit Integration
- **Address Recording**: Record socket addresses for audit trails
- **Operation Logging**: Log socket operations for compliance
- **Security Events**: Generate security-relevant events
- **Forensic Support**: Support forensic analysis requirements

### Capability Checks
- **CAP_NET_ADMIN**: Administrative network operations
- **CAP_NET_BIND_SERVICE**: Bind to privileged ports
- **CAP_NET_RAW**: Raw socket access
- **CAP_SYS_ADMIN**: System administration capabilities

## eBPF Integration

### BPF Hooks
```c
__weak noinline int update_socket_protocol(int family, int type, int protocol)
```

**eBPF Attachment Points**:
- **Socket Creation**: Modify socket parameters during creation
- **Option Processing**: Filter and modify socket options
- **Message Processing**: Intercept and modify messages
- **Address Resolution**: Modify address resolution behavior

### Cgroup Integration
- **Socket Option Override**: eBPF programs can override socket options
- **Traffic Control**: Integration with cgroup-based traffic control
- **Resource Limits**: Enforce cgroup-based resource limits
- **Policy Enforcement**: Implement custom networking policies

## Error Handling and Edge Cases

### Common Error Conditions
- **EBADF**: Bad file descriptor
- **ENOTSOCK**: File descriptor is not a socket
- **EAFNOSUPPORT**: Address family not supported
- **EPROTONOSUPPORT**: Protocol not supported
- **ENOMEM**: Out of memory
- **EINVAL**: Invalid argument
- **EACCES**: Permission denied

### Resource Management
- **Memory Leak Prevention**: Comprehensive cleanup on error paths
- **Reference Leak Prevention**: Proper reference counting management
- **File Descriptor Leaks**: Careful file descriptor management
- **Module Reference Cleanup**: Proper module reference handling

### Race Condition Handling
- **Socket State Races**: Handle concurrent state changes
- **File Descriptor Races**: Handle concurrent fd operations
- **Module Unload Races**: Prevent use-after-unload scenarios
- **Protocol Switch Races**: Handle protocol operation changes

## Modern Features and Extensions

### io_uring Integration
```c
.uring_cmd = io_uring_cmd_sock,
```

**Asynchronous Operations**:
- **Command Interface**: Direct socket commands through io_uring
- **Zero-Copy I/O**: Efficient asynchronous data transfer
- **Batched Operations**: Efficient handling of multiple operations
- **Event Notification**: Integration with io_uring event model

### Container and Namespace Support
- **Network Namespaces**: Isolated networking environments
- **Mount Namespaces**: Isolated filesystem views
- **User Namespaces**: User ID mapping and isolation
- **Container Integration**: Support for containerized applications

### High-Performance Features
- **BUSY_POLL**: Reduced latency through busy polling
- **SO_REUSEPORT**: Load balancing across multiple sockets
- **TCP_FASTOPEN**: Reduced connection establishment overhead
- **UDP_GRO**: Generic receive offload for UDP

## Protocol Family Management

### Dynamic Protocol Loading
```c
#ifdef CONFIG_MODULES
if (rcu_access_pointer(net_families[family]) == NULL)
    request_module("net-pf-%d", family);
#endif
```

**Module Management**:
- **Automatic Loading**: Load protocol modules on demand
- **Reference Counting**: Track module usage
- **Safe Unloading**: Prevent unsafe module unloading
- **Protocol Registration**: Dynamic protocol family registration

### Protocol Family Array
```c
static const struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;
```

**Family Management**:
- **RCU Protection**: Safe concurrent access to protocol families
- **Dynamic Registration**: Runtime protocol family registration
- **Module Integration**: Link protocols to their owning modules

## Debugging and Monitoring

### Tracing Support
- **Filesystem Events**: Trace socket filesystem operations
- **System Call Tracing**: Trace socket system calls
- **Performance Monitoring**: Monitor socket performance metrics
- **Security Events**: Trace security-relevant operations

### Statistics and Metrics
- **Socket Counters**: Track socket creation and destruction
- **Error Counters**: Monitor error frequencies
- **Performance Metrics**: Track operation latencies
- **Memory Usage**: Monitor socket memory consumption

### Debugging Features
- **Socket Information**: Expose socket state through /proc and debugfs
- **Error Reporting**: Comprehensive error reporting
- **Validation**: Runtime validation of socket state
- **Assertions**: Debug assertions for development

This implementation represents the cornerstone of Linux networking, providing a robust, secure, and high-performance interface that bridges user applications with the kernel's networking subsystem. It balances simplicity of use with powerful features, supporting everything from simple client-server applications to high-performance network services and modern containerized environments.