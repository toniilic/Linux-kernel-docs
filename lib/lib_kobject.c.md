# lib/kobject.c - Linux Kernel Object Infrastructure

## Overview

This file implements the foundational object-oriented infrastructure for the Linux kernel, providing unified object lifecycle management, hierarchical organization, reference counting, sysfs integration, and event notification systems. Originally developed as part of the Linux device model, the kobject system has evolved into a sophisticated foundation supporting device management, container isolation, and kernel-to-userspace communication through uevents.

## Historical Development

### Key Evolution Points
- **Early 2000s**: Introduction as part of the device model framework
- **2005-2010**: Sysfs integration and hierarchical organization features
- **2010s**: Container and namespace support for virtualization
- **Modern Era**: Advanced debugging, security hardening, and uevent system maturity

### Design Philosophy
The kobject infrastructure embodies object-oriented design principles in kernel space, providing polymorphism through type systems, automatic memory management through reference counting, and unified interfaces for object representation and lifecycle management while maintaining performance and security.

## Core Architecture

### Object Model Pipeline
```
Object Creation → Initialization → Hierarchy Integration → Sysfs Representation → Event Notification → Cleanup
      ↓              ↓               ↓                     ↓                      ↓               ↓
[kobject_create] [kobject_init] [kobject_add]        [create_dir]         [kobject_uevent]  [kobject_cleanup]
```

### Type System Architecture
1. **Polymorphic Behavior**: Objects define behavior through kobj_type function pointers
2. **Resource Management**: Type-specific cleanup through release callbacks
3. **Sysfs Integration**: Automatic attribute group creation and file operations
4. **Namespace Support**: Container-aware object visibility and security

## Key Data Structures

### Core Object Structure
```c
struct kobject {
    const char        *name;           /* Object name */
    struct list_head   entry;          /* List membership in kset */
    struct kobject    *parent;         /* Hierarchical parent */
    struct kset       *kset;           /* Container set */
    const struct kobj_type *ktype;     /* Type operations */
    struct kernfs_node *sd;            /* sysfs directory entry */
    struct kref        kref;           /* Reference counter */
    
    /* State tracking bitfields */
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
    
    #ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    struct delayed_work release;       /* Debug delayed cleanup */
    #endif
};
```

### Type System Definition
```c
struct kobj_type {
    void (*release)(struct kobject *kobj);                    /* Destructor */
    const struct sysfs_ops *sysfs_ops;                       /* sysfs operations */
    const struct attribute_group **default_groups;           /* Default attributes */
    const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *kobj);
    const void *(*namespace)(const struct kobject *kobj);    /* Namespace support */
    void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

### Container Organization
```c
struct kset {
    struct list_head list;                        /* List of member kobjects */
    spinlock_t list_lock;                        /* Protection for list operations */
    struct kobject kobj;                         /* Embedded kobject (recursive) */
    const struct kset_uevent_ops *uevent_ops;   /* Event handling operations */
};
```

### UEvent Infrastructure
```c
struct kobj_uevent_env {
    char *argv[3];                    /* For usermode helper */
    char *envp[UEVENT_NUM_ENVP];     /* Environment variable pointers (64 max) */
    int envp_idx;                     /* Current index */
    char buf[UEVENT_BUFFER_SIZE];    /* String storage buffer (2048 bytes) */
    int buflen;                       /* Current buffer length */
};
```

## Core Functions

### Object Lifecycle Management

#### `kobject_init()` - Object Initialization
```c
void kobject_init(struct kobject *kobj, const struct kobj_type *ktype)
```

**Purpose**: Initialize a kobject structure with a specific type

**Security and Validation**:
```c
void kobject_init(struct kobject *kobj, const struct kobj_type *ktype)
{
    /* Parameter validation with error handling */
    if (!kobj || !ktype) {
        pr_err("kobject (%p): %s\n", kobj, err_str);
        dump_stack_lvl(KERN_ERR);
        return;
    }
    
    /* Detect double initialization */
    if (kobj->state_initialized) {
        pr_err("kobject (%p): tried to init an initialized object\n", kobj);
        dump_stack_lvl(KERN_ERR);
    }
    
    kobject_init_internal(kobj);  /* Set up internal state */
    kobj->ktype = ktype;          /* Associate type operations */
}
```

**Initialization Process**:
1. **Reference counting**: Initializes kref with count = 1
2. **List membership**: Initializes list entry for kset membership
3. **State tracking**: Sets `state_initialized = 1`, others = 0
4. **Type association**: Links object to its behavioral type

#### `kobject_add()` - Hierarchy Integration
```c
int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...)
```

**Purpose**: Add an initialized kobject to the hierarchy and sysfs

**Parent Resolution Logic**:
1. **Explicit parent**: If parent provided, use it directly
2. **Kset fallback**: If no parent but kset exists, use kset's kobject as parent
3. **Root placement**: Otherwise, object becomes root-level

**Integration Process**:
```c
static int kobject_add_internal(struct kobject *kobj)
{
    struct kobject *parent;
    
    parent = kobject_get(kobj->parent);  /* Explicit parent takes precedence */
    
    if (kobj->kset) {
        if (!parent)
            parent = kobject_get(&kobj->kset->kobj);  /* Kset as fallback parent */
        kobj_kset_join(kobj);  /* Add to kset's object list */
        kobj->parent = parent;
    }
    
    error = create_dir(kobj);  /* Create sysfs representation */
    if (error) {
        kobj_kset_leave(kobj);
        kobject_put(parent);
        kobj->parent = NULL;
        return error;
    }
    
    kobj->state_in_sysfs = 1;
    return 0;
}
```

#### `kobject_create()` - Dynamic Allocation
```c
static struct kobject *kobject_create(void)
```

**Purpose**: Dynamically allocate and initialize a kobject

**Implementation**:
```c
static struct kobject *kobject_create(void)
{
    struct kobject *kobj;
    
    kobj = kzalloc(sizeof(*kobj), GFP_KERNEL);
    if (!kobj)
        return NULL;
        
    kobject_init(kobj, &dynamic_kobj_ktype);  /* Uses default dynamic type */
    return kobj;
}
```

### Reference Counting System

#### `kobject_get()` - Safe Reference Increment
```c
struct kobject *kobject_get(struct kobject *kobj)
```

**Security Features**:
```c
struct kobject *kobject_get(struct kobject *kobj)
{
    if (kobj) {
        if (!kobj->state_initialized)
            WARN(1, "kobject: '%s' (%p): is not initialized, yet kobject_get() is being called.\n",
                 kobject_name(kobj), kobj);
        kref_get(&kobj->kref);  /* Atomic increment */
    }
    return kobj;
}
```

#### `kobject_put()` - Reference Decrement and Cleanup Trigger
```c
void kobject_put(struct kobject *kobj)
```

**Cleanup Orchestration**:
```c
void kobject_put(struct kobject *kobj)
{
    if (kobj) {
        if (!kobj->state_initialized)
            WARN(1, "kobject: '%s' (%p): is not initialized, yet kobject_put() is being called.\n",
                 kobject_name(kobj), kobj);
        kref_put(&kobj->kref, kobject_release);  /* Calls kobject_release when count hits 0 */
    }
}
```

#### `kobject_get_unless_zero()` - Race-Safe Reference Acquisition
```c
struct kobject *kobject_get_unless_zero(struct kobject *kobj)
```

**Purpose**: Safe increment for lockless lookup patterns

**Race Prevention**:
```c
struct kobject *kobject_get_unless_zero(struct kobject *kobj)
{
    if (!kobj)
        return NULL;
    if (!kref_get_unless_zero(&kobj->kref))
        kobj = NULL;
    return kobj;
}
```

### Memory Management and Cleanup

#### `kobject_release()` - Reference Count Zero Handler
```c
static void kobject_release(struct kref *kref)
```

**Debug and Production Modes**:
```c
static void kobject_release(struct kref *kref)
{
    struct kobject *kobj = container_of(kref, struct kobject, kref);
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    unsigned long delay = HZ + HZ * get_random_u32_below(4);
    /* Delayed cleanup for debugging use-after-free */
    schedule_delayed_work(&kobj->release, delay);
#else
    kobject_cleanup(kobj);
#endif
}
```

#### `kobject_cleanup()` - Comprehensive Resource Cleanup
```c
static void kobject_cleanup(struct kobject *kobj)
```

**Cleanup Sequence**:
1. **Safety Validation**: Warns if no release function provided
2. **Sysfs Cleanup**: Removes from sysfs if still present
3. **Type-Specific Release**: Calls ktype's release function
4. **Memory Cleanup**: Frees allocated name string
5. **Parent Reference**: Drops parent reference last for hierarchy integrity

```c
static void kobject_cleanup(struct kobject *kobj)
{
    const struct kobj_type *t = get_ktype(kobj);
    
    /* Safety checks */
    if (t && !t->release)
        pr_debug("kobject: '%s' (%p): does not have a release() function\n",
                 kobject_name(kobj), kobj);
    
    /* Remove from sysfs if still present */
    if (kobj->state_in_sysfs)
        __kobject_del(kobj);
    
    /* Call type-specific cleanup */
    if (t && t->release) {
        t->release(kobj);
    }
    
    /* Free allocated name */
    kfree_const(kobj->name);
    
    /* Drop parent reference last */
    kobject_put(kobj->parent);
}
```

### Naming and Hierarchy Management

#### `kobject_set_name()` - Dynamic Name Assignment
```c
int kobject_set_name(struct kobject *kobj, const char *fmt, ...)
```

**Security Implementation**:
```c
int kobject_set_name_vargs(struct kobject *kobj, const char *fmt, va_list vargs)
{
    const char *s;
    
    if (kobj->name && !fmt)
        return 0;
    
    s = kvasprintf_const(GFP_KERNEL, fmt, vargs);  /* Smart allocation */
    if (!s)
        return -ENOMEM;
    
    /* Handle '/' character replacement for sysfs compatibility */
    if (strchr(s, '/')) {
        char *t = kstrdup(s, GFP_KERNEL);
        kfree_const(s);
        if (!t) return -ENOMEM;
        s = strreplace(t, '/', '!');  /* Replace '/' with '!' */
    }
    
    kfree_const(kobj->name);  /* Free old name */
    kobj->name = s;           /* Assign new name */
    return 0;
}
```

#### `kobject_rename()` - Atomic Renaming with Sysfs Updates
```c
int kobject_rename(struct kobject *kobj, const char *new_name)
```

**Atomic Renaming Process**:
1. **Path Capture**: Save old path for uevent notification
2. **Name Preparation**: Duplicate new name for atomic operation
3. **Sysfs Update**: Update sysfs directory name atomically
4. **Object Update**: Update kobject name only after successful sysfs update
5. **Event Notification**: Send KOBJ_MOVE uevent with old path information

#### `kobject_get_path()` - Hierarchical Path Construction
```c
char *kobject_get_path(const struct kobject *kobj, gfp_t gfp_mask)
```

**Path Building Algorithm**:
```c
static int fill_kobj_path(const struct kobject *kobj, char *path, int length)
{
    const struct kobject *parent;
    
    --length;  /* Reserve space for null terminator */
    for (parent = kobj; parent; parent = parent->parent) {
        int cur = strlen(kobject_name(parent));
        length -= cur;
        if (length <= 0) return -EINVAL;  /* Buffer overflow protection */
        
        memcpy(path + length, kobject_name(parent), cur);
        *(path + --length) = '/';  /* Add path separator */
    }
    return 0;
}
```

### Container Organization

#### `kobj_kset_join()` - Thread-Safe Set Membership
```c
static void kobj_kset_join(struct kobject *kobj)
```

**Thread Safety Implementation**:
```c
static void kobj_kset_join(struct kobject *kobj)
{
    if (!kobj->kset)
        return;
        
    kset_get(kobj->kset);                    /* Increment kset reference */
    spin_lock(&kobj->kset->list_lock);
    list_add_tail(&kobj->entry, &kobj->kset->list);  /* Add to kset's list */
    spin_unlock(&kobj->kset->list_lock);
}
```

#### `kset_find_obj()` - Safe Object Lookup
```c
struct kobject *kset_find_obj(struct kset *kset, const char *name)
```

**Race-Safe Lookup**:
```c
struct kobject *kset_find_obj(struct kset *kset, const char *name)
{
    struct kobject *k;
    struct kobject *ret = NULL;
    
    spin_lock(&kset->list_lock);
    list_for_each_entry(k, &kset->list, entry) {
        if (kobject_name(k) && !strcmp(kobject_name(k), name)) {
            ret = kobject_get_unless_zero(k);  /* Safe reference acquisition */
            break;
        }
    }
    spin_unlock(&kset->list_lock);
    return ret;
}
```

### Sysfs Integration

#### `create_dir()` - Sysfs Directory Creation
```c
static int create_dir(struct kobject *kobj)
```

**Comprehensive Directory Setup**:
```c
static int create_dir(struct kobject *kobj)
{
    const struct kobj_type *ktype = get_ktype(kobj);
    const struct kobj_ns_type_operations *ops;
    int error;
    
    error = sysfs_create_dir_ns(kobj, kobject_namespace(kobj));
    if (error)
        return error;

    if (ktype) {
        error = sysfs_create_groups(kobj, ktype->default_groups);
        if (error) {
            sysfs_remove_dir(kobj);
            return error;
        }
    }
    
    /* Namespace enablement for child object filtering */
    ops = kobj_child_ns_ops(kobj);
    if (ops) {
        BUG_ON(!kobj_ns_type_is_valid(ops->type));
        BUG_ON(!kobj_ns_type_registered(ops->type));
        sysfs_enable_ns(kobj->sd);
    }
    
    return 0;
}
```

### UEvent System

#### `kobject_uevent_env()` - Event Broadcasting
```c
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action, char *envp_ext[])
```

**Event Processing Pipeline**:
1. **Event Validation**: Validates kobject and action type
2. **Filtering**: Multiple filtering layers (object, subsystem, namespace)
3. **Environment Construction**: Builds environment variables
4. **Network Broadcasting**: Sends via netlink sockets
5. **Legacy Support**: Usermode helper fallback

**Environment Variable Construction**:
```c
/* Standard environment variables */
add_uevent_var(env, "ACTION=%s", action_string);
add_uevent_var(env, "DEVPATH=%s", devpath);
add_uevent_var(env, "SUBSYSTEM=%s", subsystem);
add_uevent_var(env, "SEQNUM=%llu", ++uevent_seqnum);
```

**Network Broadcasting Implementation**:
```c
/* Netlink message construction */
static struct sk_buff *alloc_uevent_skb(struct kobj_uevent_env *env,
                                        const char *action_string,
                                        const char *devpath)
{
    /* Message format: action_string@devpath\0ENV_VAR=value\0... */
    sprintf(scratch, "%s@%s", action_string, devpath);
    skb_put_data(skb, env->buf, env->buflen);
    return skb;
}
```

### Namespace and Security Features

#### Namespace Operations
```c
const struct kobj_ns_type_operations {
    enum kobj_ns_type type;
    bool (*current_may_mount)(void);
    void *(*grab_current_ns)(void);
    const void *(*netlink_ns)(struct sock *sk);
    const void *(*initial_ns)(void);
    void (*drop_ns)(void *);
};
```

#### Security Integration
```c
void kobject_get_ownership(const struct kobject *kobj, kuid_t *uid, kgid_t *gid)
{
    *uid = GLOBAL_ROOT_UID;
    *gid = GLOBAL_ROOT_GID;
    if (kobj->ktype->get_ownership)
        kobj->ktype->get_ownership(kobj, uid, gid);
}
```

**Input Validation and Security**:
1. **Name sanitization**: Replaces '/' with '!' to prevent directory traversal
2. **Reference counting safety**: Atomic operations prevent race conditions
3. **Memory allocation failure handling**: Comprehensive error recovery
4. **State validation**: Prevents double-initialization and improper usage
5. **Namespace isolation**: Container-aware object visibility

## Advanced Features

### Debug Support
- **CONFIG_DEBUG_KOBJECT_RELEASE**: Random cleanup delays to expose use-after-free bugs
- **State tracking**: Multiple flags prevent double-operations
- **Comprehensive logging**: Detailed debug information for troubleshooting

### Container Integration
- **Namespace awareness**: Full support for network, user, and other namespaces
- **Event filtering**: Namespace-specific event delivery
- **Credential mapping**: Proper UID/GID translation for containers

### Performance Optimizations
- **Reference counting**: Atomic operations for thread safety
- **Spinlock protection**: Minimal lock granularity for kset operations
- **Memory efficiency**: Const-aware allocation and deallocation

## Integration Points

### Device Model Integration
- **Foundation for device structures**: All kernel devices built on kobjects
- **Bus and driver systems**: Hierarchical organization through kobject parent-child relationships
- **Power management**: Integration with PM core through object hierarchy

### Sysfs Filesystem Integration
- **Automatic directory creation**: Objects automatically appear in sysfs
- **Attribute group support**: Declarative sysfs file creation
- **Permission management**: Security-aware file ownership and permissions

### Event System Integration
- **Netlink socket communication**: Kernel-to-userspace event delivery
- **Udev integration**: Dynamic device management support
- **Container event filtering**: Namespace-aware event delivery

This comprehensive kobject implementation provides the foundational object-oriented infrastructure for the Linux kernel, demonstrating sophisticated reference counting, hierarchical organization, security features, and integration with userspace through sysfs and uevents. The system successfully balances performance, security, and functionality while serving as the basis for the entire Linux device model.