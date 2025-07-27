# drivers/base/core.c - Linux Device Driver Model Core Implementation

## Overview

This file implements the core device driver model for the Linux kernel. Originally designed by Patrick Mochel and enhanced by Greg Kroah-Hartman, it provides the fundamental infrastructure for device registration, management, and interaction between devices and drivers. The implementation serves as the foundation for all device management in Linux, from simple platform devices to complex PCI and USB subsystems.

## Historical Development

### Key Contributors and Evolution
- **Patrick Mochel (2002-2003)**: Original device driver model design and implementation
- **Greg Kroah-Hartman (2006-present)**: Major enhancements and ongoing maintenance
- **Open Source Development Labs**: Early development support
- **Novell Inc.**: Additional corporate contributions

### Major Milestones
- **2002**: Initial device driver model implementation
- **2004**: sysfs integration and device hierarchy
- **2006**: Major refactoring and stabilization
- **2010s**: Device links and dependency management
- **2020s**: Modern features including fw_devlink and advanced power management

## Core Concepts

### Device Driver Model Architecture

#### Device Hierarchy
The Linux device model organizes devices in a hierarchical tree structure:
- **Root Level**: Virtual devices and platform buses
- **Bus Level**: PCI, USB, I2C, SPI buses
- **Device Level**: Individual devices on buses
- **Child Devices**: Composite devices with subcomponents

#### Object-Oriented Design
- **Inheritance**: Devices inherit from kobject infrastructure
- **Polymorphism**: Bus-specific operations through function pointers
- **Encapsulation**: Private device data and controlled access
- **Reference Counting**: Automatic memory management through kobjects

### Device States and Lifecycle
```
    [Created] → [Initialized] → [Added] → [Probed] → [Bound] → [Active]
                     ↓              ↓         ↓        ↓         ↓
    [Destroyed] ← [Removed] ← [Unbound] ← [Released] ← [Offline]
```

## Key Data Structures

### `struct device` - Core Device Structure
```c
struct device {
    struct device           *parent;        /* Parent device */
    struct device_private   *p;             /* Private data */
    struct kobject          kobj;           /* Kernel object */
    const char              *init_name;     /* Initial name */
    const struct device_type *type;        /* Device type */
    struct bus_type         *bus;           /* Bus type */
    struct device_driver    *driver;       /* Associated driver */
    void                    *platform_data; /* Platform specific data */
    void                    *driver_data;   /* Driver specific data */
    struct dev_links_info   links;          /* Device links */
    struct dev_pm_info      power;          /* Power management */
    u64                     *dma_mask;      /* DMA mask */
    u64                     coherent_dma_mask; /* Coherent DMA mask */
    unsigned long           dma_pools;      /* DMA pools */
    struct dma_coherent_mem *dma_mem;       /* Coherent DMA memory */
    struct cma              *cma_area;      /* CMA area */
    dev_t                   devt;           /* Device number */
    u32                     id;             /* Device instance ID */
    spinlock_t              devres_lock;    /* Device resource lock */
    struct list_head        devres_head;    /* Device resources */
    struct klist_node       knode_class;    /* Class list node */
    const struct class      *class;         /* Device class */
    const struct attribute_group **groups; /* Attribute groups */
    void (*release)(struct device *dev);    /* Release function */
    struct mutex            mutex;          /* Device mutex */
    bool                    offline_disabled; /* Offline capability */
    bool                    offline;        /* Device offline state */
    bool                    of_node_reused; /* Device tree reuse */
    bool                    state_synced;   /* State synchronized */
    bool                    can_match;      /* Can match driver */
};
```

### `struct device_private` - Private Device Data
```c
struct device_private {
    struct klist klist_children;       /* Child device list */
    struct klist_node knode_parent;    /* Parent list node */
    struct klist_node knode_driver;    /* Driver list node */
    struct klist_node knode_bus;       /* Bus list node */
    struct klist_node knode_class;     /* Class list node */
    struct list_head deferred_probe;   /* Deferred probe list */
    struct device_driver *async_driver; /* Async driver */
    char *deferred_probe_reason;       /* Probe deferral reason */
    struct device *device;             /* Back pointer */
};
```

### `struct dev_links_info` - Device Links Management
```c
struct dev_links_info {
    struct list_head suppliers;        /* Supplier device links */
    struct list_head consumers;        /* Consumer device links */
    struct list_head defer_sync;       /* Deferred sync list */
    enum dl_dev_state status;          /* Device link status */
};
```

### Device Link States
```c
enum device_link_state {
    DL_STATE_NONE = -1,                /* No link */
    DL_STATE_DORMANT = 0,              /* Link inactive */
    DL_STATE_AVAILABLE,                /* Supplier available */
    DL_STATE_CONSUMER_PROBE,           /* Consumer probing */
    DL_STATE_ACTIVE,                   /* Link active */
    DL_STATE_SUPPLIER_UNBIND,          /* Supplier unbinding */
};
```

## Core Functions

### Device Lifecycle Management

#### `device_initialize()` - Device Structure Initialization
```c
void device_initialize(struct device *dev)
```

**Purpose**: Initialize device structure for use in the driver model

**Initialization Process**:
1. **Kobject Setup**: Initialize embedded kobject with device_ktype
2. **Reference Counting**: Set up get_device()/put_device() infrastructure
3. **List Initialization**: Initialize various device lists and locks
4. **Power Management**: Initialize power management structures
5. **DMA Setup**: Initialize DMA-related fields and coherency settings
6. **Device Links**: Initialize supplier/consumer link lists
7. **Resource Management**: Set up device resource management

**Key Operations**:
```c
dev->kobj.kset = devices_kset;
kobject_init(&dev->kobj, &device_ktype);
INIT_LIST_HEAD(&dev->dma_pools);
mutex_init(&dev->mutex);
lockdep_set_novalidate_class(&dev->mutex);
spin_lock_init(&dev->devres_lock);
INIT_LIST_HEAD(&dev->devres_head);
device_pm_init(dev);
INIT_LIST_HEAD(&dev->links.consumers);
INIT_LIST_HEAD(&dev->links.suppliers);
```

#### `device_add()` - Add Device to System
```c
int device_add(struct device *dev)
```

**Purpose**: Add initialized device to the device hierarchy and make it visible

**Registration Process**:
1. **Name Assignment**: Set device name if not already set
2. **Parent Relationship**: Establish parent-child relationships
3. **Kobject Integration**: Add to sysfs hierarchy
4. **Platform Notification**: Notify platform-specific code
5. **Attribute Creation**: Create sysfs attributes
6. **Bus Integration**: Add to appropriate bus
7. **Class Integration**: Add to device class
8. **Power Management**: Register with power management
9. **Device Links**: Process firmware device links
10. **Driver Probing**: Trigger driver matching and probing

**Error Handling**: Comprehensive rollback on any failure step

#### `device_del()` - Remove Device from System
```c
void device_del(struct device *dev)
```

**Removal Process**:
1. **Driver Unbinding**: Unbind any attached driver
2. **Link Cleanup**: Remove device links
3. **Attribute Removal**: Remove sysfs attributes
4. **Bus Removal**: Remove from bus
5. **Class Removal**: Remove from class
6. **Power Management**: Unregister from power management
7. **Kobject Removal**: Remove from sysfs hierarchy
8. **Notification**: Send removal notifications

### Device Registration Interface

#### `device_register()` - Complete Device Registration
```c
int device_register(struct device *dev)
```

**Combined Operation**:
```c
device_initialize(dev);
return device_add(dev);
```

**Usage Pattern**: Most common interface for simple device registration

#### `device_unregister()` - Complete Device Unregistration
```c
void device_unregister(struct device *dev)
```

**Combined Operation**:
```c
device_del(dev);
put_device(dev);
```

### Device Links and Dependencies

#### Device Link Types
- **DL_FLAG_MANAGED**: Managed by driver core
- **DL_FLAG_AUTOREMOVE_CONSUMER**: Remove when consumer unbinds
- **DL_FLAG_AUTOREMOVE_SUPPLIER**: Remove when supplier unbinds
- **DL_FLAG_AUTOPROBE_CONSUMER**: Trigger consumer reprobe
- **DL_FLAG_SYNC_STATE_ONLY**: Synchronization only
- **DL_FLAG_INFERRED**: Inferred from firmware

#### `device_link_add()` - Create Device Dependency
```c
struct device_link *device_link_add(struct device *consumer,
                                   struct device *supplier,
                                   u32 flags)
```

**Link Management**:
1. **Validation**: Verify consumer and supplier validity
2. **Duplicate Check**: Prevent duplicate links
3. **State Synchronization**: Set appropriate link state
4. **Reference Management**: Take device references
5. **List Management**: Add to consumer/supplier lists

#### `device_links_check_suppliers()` - Verify Dependencies
```c
int device_links_check_suppliers(struct device *dev)
```

**Dependency Verification**:
1. **Supplier Scanning**: Check all supplier dependencies
2. **State Validation**: Verify suppliers are available
3. **Probe Deferral**: Return -EPROBE_DEFER if suppliers not ready
4. **Link State Update**: Update link states appropriately

### Firmware Device Links (fw_devlink)

#### `fwnode_link_add()` - Add Firmware Node Link
```c
int fwnode_link_add(struct fwnode_handle *con, struct fwnode_handle *sup, u8 flags)
```

**Firmware Dependency Management**:
- **Device Tree**: Process device tree dependencies
- **ACPI**: Handle ACPI device dependencies
- **Automatic Creation**: Create device links from firmware data
- **Cycle Detection**: Detect and handle dependency cycles

#### fw_devlink States
- **Permissive**: Allow all device creation
- **Best Effort**: Try to enforce dependencies but allow fallback
- **Strict**: Enforce all firmware dependencies

### Sysfs Integration

#### `device_create_file()` - Create Device Attribute
```c
int device_create_file(struct device *dev, const struct device_attribute *attr)
```

**Attribute Management**:
- **Permission Validation**: Verify read/write permissions match functions
- **Sysfs Creation**: Create sysfs attribute file
- **Error Handling**: Handle creation failures gracefully

#### Device Attribute Structure
```c
struct device_attribute {
    struct attribute attr;              /* Base attribute */
    ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
    ssize_t (*store)(struct device *dev, struct device_attribute *attr, const char *buf, size_t count);
};
```

### Device Hierarchy and Parent-Child Relationships

#### `device_for_each_child()` - Child Device Iteration
```c
int device_for_each_child(struct device *parent, void *data, device_iter_t fn)
```

**Child Management**:
- **Safe Iteration**: Thread-safe child device iteration
- **Reference Counting**: Proper reference management during iteration
- **Callback Processing**: Apply function to each child device
- **Early Termination**: Support for early iteration termination

#### `device_find_child()` - Locate Child Device
```c
struct device *device_find_child(struct device *parent, const void *data, device_match_t match)
```

**Child Location**:
- **Pattern Matching**: Find child based on match function
- **Reference Return**: Return reference to found child
- **Safe Access**: Thread-safe child access

### Resource Management (devres)

#### Managed Resource Framework
```c
void *devm_kmalloc(struct device *dev, size_t size, gfp_t gfp);
void devm_kfree(struct device *dev, void *p);
```

**Automatic Cleanup**:
- **Memory Management**: Automatic memory cleanup on device removal
- **Resource Tracking**: Track all device-managed resources
- **Error Handling**: Automatic cleanup on probe failures
- **Driver Simplification**: Reduce driver cleanup code

#### Resource Types
- **Memory**: devm_kmalloc, devm_kzalloc
- **I/O**: devm_ioremap, devm_request_region
- **Interrupts**: devm_request_irq
- **Clocks**: devm_clk_get
- **Regulators**: devm_regulator_get
- **GPIO**: devm_gpio_request

### Power Management Integration

#### Device Power States
- **Active**: Device fully operational
- **Runtime Suspended**: Device temporarily suspended for power saving
- **System Suspended**: Device suspended for system suspend
- **Off**: Device powered off

#### `device_pm_add()` - Add to Power Management
```c
void device_pm_add(struct device *dev)
```

**Power Management Registration**:
- **PM List**: Add to global device power management list
- **Runtime PM**: Initialize runtime power management
- **Wakeup Sources**: Configure wakeup capabilities
- **Power Domains**: Associate with power domains

### Device Matching and Probing

#### Driver Matching Process
1. **Bus Matching**: Bus-specific match function
2. **Device Tree**: Device tree compatible strings
3. **ACPI**: ACPI device IDs
4. **Platform**: Platform device names
5. **Generic**: Generic device/driver matching

#### `bus_probe_device()` - Trigger Device Probing
```c
void bus_probe_device(struct device *dev)
```

**Probing Process**:
1. **Driver Search**: Search for compatible driver
2. **Match Verification**: Verify driver compatibility
3. **Probe Execution**: Execute driver probe function
4. **Error Handling**: Handle probe failures and deferrals
5. **State Update**: Update device and link states

### Device Offline/Online Support

#### `device_offline()` - Prepare Device for Removal
```c
int device_offline(struct device *dev)
```

**Hot-Remove Preparation**:
1. **Child Check**: Verify all children can be offlined
2. **Driver Notification**: Notify driver of offline request
3. **State Update**: Update device offline state
4. **Resource Quiesce**: Quiesce device resources

#### `device_online()` - Bring Device Back Online
```c
int device_online(struct device *dev)
```

**Online Restoration**:
1. **Capability Check**: Verify device can be brought online
2. **Driver Notification**: Notify driver of online request
3. **State Restoration**: Restore device to active state
4. **Resource Activation**: Reactivate device resources

## Advanced Features

### Deferred Probing

#### Mechanism
- **EPROBE_DEFER**: Special return code for probe deferral
- **Retry Queue**: Queue of devices waiting for dependencies
- **Automatic Retry**: Automatic reprobe when dependencies available
- **Timeout Handling**: Handle indefinite deferrals gracefully

#### `device_links_check_suppliers()` Integration
- **Dependency Validation**: Check supplier availability
- **Automatic Deferral**: Automatically defer if suppliers not ready
- **Link State Management**: Update link states appropriately

### Device Links and Ordering

#### Probe Ordering
- **Supplier First**: Ensure suppliers probe before consumers
- **Dependency Chains**: Handle complex dependency chains
- **Cycle Breaking**: Break dependency cycles when necessary
- **Best Effort Mode**: Allow probing with missing dependencies

#### Unbind Ordering
- **Consumer First**: Unbind consumers before suppliers
- **Graceful Degradation**: Handle supplier removal gracefully
- **State Synchronization**: Maintain consistent device states

### Modern Extensions

#### Firmware Device Links (fw_devlink)
- **Device Tree Support**: Extract dependencies from device tree
- **ACPI Support**: Handle ACPI device dependencies
- **Automatic Management**: Automatic device link creation
- **Boot Optimization**: Optimize boot-time device probing

#### Sync State Framework
- **State Synchronization**: Coordinate device state across dependencies
- **Late Initialization**: Handle initialization after all consumers ready
- **Power Optimization**: Optimize power states based on usage
- **Resource Management**: Manage shared resources efficiently

## Error Handling and Recovery

### Device Registration Failures
- **Partial Registration**: Handle partial registration cleanup
- **Resource Leaks**: Prevent resource leaks on failure
- **State Consistency**: Maintain consistent device states
- **Error Propagation**: Proper error code propagation

### Runtime Error Handling
- **Device Removal**: Handle hot device removal
- **Driver Failures**: Handle driver probe/remove failures
- **Resource Exhaustion**: Handle resource allocation failures
- **State Corruption**: Detect and handle state corruption

### Recovery Mechanisms
- **Automatic Retry**: Retry failed operations when appropriate
- **Graceful Degradation**: Continue operation with reduced functionality
- **Error Reporting**: Comprehensive error reporting and logging
- **Debugging Support**: Rich debugging and tracing capabilities

## Performance Optimizations

### Lock Optimization
- **Fine-Grained Locking**: Minimize lock contention
- **RCU Integration**: Use RCU for read-heavy operations
- **Lock Ordering**: Prevent deadlocks through consistent ordering
- **Lockless Operations**: Use lockless algorithms where possible

### Memory Management
- **Slab Caching**: Efficient allocation for common objects
- **Reference Counting**: Minimal overhead reference counting
- **Memory Pooling**: Pool frequently allocated objects
- **Lazy Cleanup**: Defer expensive cleanup operations

### Scalability Features
- **Parallel Probing**: Allow parallel device probing where safe
- **Hierarchical Organization**: Efficient device tree organization
- **Indexed Lookup**: Fast device lookup mechanisms
- **Batch Operations**: Batch multiple operations for efficiency

## Security and Safety

### Access Control
- **Permission Checking**: Validate device access permissions
- **Capability Requirements**: Require appropriate capabilities
- **Sandboxing**: Support for device access sandboxing
- **Audit Support**: Comprehensive audit trail

### Resource Protection
- **Resource Isolation**: Isolate device resources between users
- **DMA Protection**: Protect against malicious DMA operations
- **Address Space Isolation**: Isolate device address spaces
- **Secure Boot**: Support for secure boot device verification

### Safety Mechanisms
- **Reference Validation**: Validate all device references
- **State Verification**: Verify device state consistency
- **Bounds Checking**: Check all array and pointer accesses
- **Overflow Protection**: Protect against buffer overflows

## Integration Points

### Bus Subsystem Integration
- **Bus Registration**: Register with appropriate bus subsystems
- **Driver Matching**: Integration with driver matching mechanisms
- **Power Management**: Coordinate with bus power management
- **Hot-Plug Support**: Support for hot-pluggable buses

### Class Subsystem Integration
- **Class Organization**: Organize devices by functional class
- **Common Interfaces**: Provide common interfaces for device classes
- **User Space API**: Expose class-specific user space APIs
- **Policy Enforcement**: Enforce class-specific policies

### Sysfs Integration
- **Hierarchical Structure**: Reflect device hierarchy in sysfs
- **Attribute Management**: Manage device-specific attributes
- **Link Management**: Handle symbolic links between related devices
- **User Space Interface**: Provide comprehensive user space interface

This comprehensive device driver model implementation provides the foundation for all device management in Linux, enabling complex device hierarchies, automatic dependency management, and robust error handling while maintaining high performance and security standards.