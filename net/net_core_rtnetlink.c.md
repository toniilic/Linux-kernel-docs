# Linux Kernel Routing Netlink Interface (`net/core/rtnetlink.c`)

## Overview

The `net/core/rtnetlink.c` file implements the kernel's routing netlink socket interface, containing 7,089 lines of critical networking infrastructure code. This file serves as the primary communication bridge between userspace network management tools (like `ip`, `bridge`, `tc`) and the kernel's networking subsystem. It provides a protocol-independent interface for managing network devices, routing tables, network addresses, neighbor entries, and various networking statistics. The rtnetlink interface is fundamental to Linux network administration, enabling configuration and monitoring of all aspects of the network stack through a unified netlink socket API.

## Core Architecture

### 1. RTNL Mutex and Locking Infrastructure

**Primary Locking System** - Lines 76-134:
```c
static DEFINE_MUTEX(rtnl_mutex);

void rtnl_lock(void)
{
    mutex_lock(&rtnl_mutex);
}
EXPORT_SYMBOL(rtnl_lock);

int __rtnl_trylock(void)
{
    return mutex_trylock(&rtnl_mutex);
}

void rtnl_unlock(void)
{
    struct sk_buff *head = defer_kfree_skb_list;
    
    defer_kfree_skb_list = NULL;
    
    /* Ensure other CPUs see updated state */
    smp_store_release(&rtnl_nets.num, 0);
    
    /* Drop skbs after releasing rtnl_mutex */
    while (head) {
        struct sk_buff *next = head->next;
        kfree_skb(head);
        cond_resched();
        head = next;
    }
    
    if (rtnl_nets.unregistering_cnt)
        dev_net_notifier_run_all(NULL);
        
    mutex_unlock(&rtnl_mutex);
    
    if (refcount_dec_and_test(&rtnl_net_generation))
        wake_up(&netdev_unregistering_wq);
}

int rtnl_lock_killable(void)
{
    return mutex_lock_killable(&rtnl_mutex);
}

bool rtnl_is_locked(void)
{
    return mutex_is_locked(&rtnl_mutex);
}
```

**RTNL Deferred Operations** - Lines 94-101:
```c
static struct sk_buff *defer_kfree_skb_list;

void rtnl_kfree_skbs(struct sk_buff *head, struct sk_buff *tail)
{
    if (head && tail) {
        tail->next = defer_kfree_skb_list;
        defer_kfree_skb_list = head;
    }
}
```

**Locking Features**:
- **Global RTNL Mutex**: Single mutex protecting all routing table modifications
- **Deferred Cleanup**: SKB cleanup deferred until lock release to avoid deadlocks
- **Interruptible Locking**: Support for killable lock operations
- **Network Namespace Management**: Coordinates network namespace operations
- **Generation Counting**: Reference counting for network generation tracking
- **Export to Modules**: Core locking functions exported for driver use

### 2. Netlink Message Handling Infrastructure

**Core Message Handler** - Lines 6845-6915:
```c
static int rtnetlink_rcv_msg(struct sk_buff *skb, struct nlmsghdr *nlh,
                struct netlink_ext_ack *extack)
{
    struct net *net = sock_net(skb->sk);
    struct rtnl_link *link;
    enum rtnl_kinds kind;
    struct module *owner;
    int err = -EOPNOTSUPP;
    rtnl_doit_func doit;
    unsigned int flags;
    int family;
    int type;

    type = nlh->nlmsg_type;
    if (type > RTM_MAX)
        return -EOPNOTSUPP;

    type -= RTM_BASE;

    /* All the messages must have at least 1 byte length */
    if (nlmsg_len(nlh) < sizeof(struct rtgenmsg))
        return 0;

    family = ((struct rtgenmsg *)nlmsg_data(nlh))->rtgen_family;
    kind = rtnl_msgtype_kind(type);

    if (kind != RTNL_KIND_GET && !netlink_net_capable(skb, CAP_NET_ADMIN))
        return -EPERM;

    rcu_read_lock();
    if (kind == RTNL_KIND_GET && (nlh->nlmsg_flags & NLM_F_DUMP)) {
        struct sock *rtnl;
        rtnl_dumpit_func dumpit;
        u32 min_dump_alloc = 0;

        link = rtnl_get_link(family, type);
        if (!link || !link->dumpit) {
            family = PF_UNSPEC;
            link = rtnl_get_link(family, type);
            if (!link || !link->dumpit)
                goto err_unlock;
        }
        owner = link->owner;
        dumpit = link->dumpit;
        flags = link->flags;

        if (type == RTM_GETLINK - RTM_BASE)
            min_dump_alloc = rtnl_calcit(skb, nlh);

        rcu_read_unlock();
        rtnl = net->rtnl;
        {
            struct netlink_dump_control c = {
                .dump		= dumpit,
                .min_dump_alloc	= min_dump_alloc,
                .module		= owner,
            };
            err = netlink_dump_start(rtnl, skb, nlh, &c);
        }
        return err;
    }

    link = rtnl_get_link(family, type);
    if (!link || !link->doit) {
        family = PF_UNSPEC;
        link = rtnl_get_link(family, type);
        if (!link || !link->doit)
            goto out_unlock;
    }

    owner = link->owner;
    if (!try_module_get(owner)) {
        err = -EPROTONOSUPPORT;
        goto out_unlock;
    }

    flags = link->flags;
    if (get_net_flag(net, NETNS_DELETE) && !(flags & RTNL_FLAG_DOIT_UNLOCKED)) {
        module_put(owner);
        err = -ENODEV;
        goto out_unlock;
    }

    doit = link->doit;
    rcu_read_unlock();

    if (doit)
        err = doit(skb, nlh, extack);

    module_put(owner);
    return err;

out_unlock:
    rcu_read_unlock();
    return err;

err_unlock:
    rcu_read_unlock();
    return -EOPNOTSUPP;
}
```

**Handler Registration Structure** - Lines 68-74:
```c
struct rtnl_link {
    rtnl_doit_func        doit;
    rtnl_dumpit_func      dumpit;
    struct module         *owner;
    unsigned int          flags;
    struct rcu_head       rcu;
};
```

**Message Handling Features**:
- **Type Validation**: Validates message types against supported range
- **Permission Checking**: Enforces CAP_NET_ADMIN for non-GET operations
- **Family Resolution**: Supports protocol family-specific handlers
- **Module Management**: Proper module reference counting for handler modules
- **RCU Protection**: RCU-safe handler lookup and execution
- **Dump Support**: Special handling for data dump operations
- **Error Handling**: Comprehensive error reporting with extended ACK

### 3. Network Device Link Management

**New Link Creation** - Lines 3944-4020:
```c
static int rtnl_newlink(struct sk_buff *skb, struct nlmsghdr *nlh,
            struct netlink_ext_ack *extack)
{
    struct net *tgt_net, *link_net = NULL, *peer_net = NULL;
    struct nlattr **tb, **linkinfo, **data = NULL;
    struct rtnl_link_ops *ops = NULL;
    struct rtnl_newlink_tbs *tbs;
    struct rtnl_nets rtnl_nets;
    int ops_srcu_index;
    int ret;

    tbs = kmalloc(sizeof(*tbs), GFP_KERNEL);
    if (!tbs)
        return -ENOMEM;

    tb = tbs->tb;
    ret = nlmsg_parse_deprecated(nlh, sizeof(struct ifinfomsg), tb,
                     IFLA_MAX, ifla_policy, extack);
    if (ret < 0)
        goto free;

    ret = rtnl_ensure_unique_netns(tb, extack, false);
    if (ret < 0)
        goto free;

    linkinfo = tbs->linkinfo;
    if (tb[IFLA_LINKINFO]) {
        ret = nla_parse_nested_deprecated(linkinfo, IFLA_INFO_MAX,
                          tb[IFLA_LINKINFO],
                          ifla_info_policy, NULL);
        if (ret < 0)
            goto free;
    } else {
        memset(linkinfo, 0, sizeof(tbs->linkinfo));
    }

    if (linkinfo[IFLA_INFO_KIND]) {
        char kind[MODULE_NAME_LEN];

        nla_strscpy(kind, linkinfo[IFLA_INFO_KIND], sizeof(kind));
        ops = rtnl_link_ops_get(kind, &ops_srcu_index);
#ifdef CONFIG_MODULES
        if (!ops) {
            request_module("rtnl-link-%s", kind);
            ops = rtnl_link_ops_get(kind, &ops_srcu_index);
        }
#endif
        if (!ops) {
            NL_SET_ERR_MSG(extack, "Unknown device type");
            ret = -EOPNOTSUPP;
            goto free;
        }
    }

    ret = rtnl_nets_init(&rtnl_nets, tb, extack, nlh);
    if (ret < 0)
        goto put_ops;

    if (ops) {
        ret = rtnl_newlink_create(skb, ifm, ops, rtnl_nets.tgt_net,
                      rtnl_nets.link_net, rtnl_nets.peer_net,
                      nlh, tb, linkinfo[IFLA_INFO_DATA], extack);
    } else {
        ret = rtnl_configure_link(dev, ifm, nlh, tb, extack);
    }

put_ops:
    if (ops)
        rtnl_link_ops_put(ops, ops_srcu_index);
    rtnl_nets_destroy(&rtnl_nets);
free:
    kfree(tbs);
    return ret;
}
```

**Link Information Dumping** - Lines 2428-2510:
```c
static int rtnl_dump_ifinfo(struct sk_buff *skb, struct netlink_callback *cb)
{
    struct netlink_ext_ack *extack = cb->extack;
    const struct nlmsghdr *nlh = cb->nlh;
    struct net *net = sock_net(skb->sk);
    struct {
        unsigned long ifindex;
        int s_h, s_idx;
    } *ctx = (void *)cb->ctx;
    struct net_device *dev;
    struct nlattr *tb[IFLA_MAX+1];
    u32 portid = NETLINK_CB(cb->skb).portid;
    u32 seq = cb->nlh->nlmsg_seq;
    struct nlattr *extfilt;
    u32 filter_mask = 0;
    int master_idx = 0;
    int netnsid = -1;
    int err, i;

    s_h = ctx->s_h;
    s_idx = ctx->s_idx;

    /* A hack to preserve kernel<->userspace interface.
     * The correct header is ifinfomsg. It is consistent with rtnl_getlink.
     * However, before Linux v3.9 the code here assumed rtgenmsg and that's
     * what iproute2 < 3.9.0 used.
     * We can detect the old iproute2. Even including the IFLA_EXT_MASK
     * attribute, its netlink message is shorter than struct ifinfomsg.
     */
    if (nlmsg_len(nlh) < sizeof(struct ifinfomsg)) {
        struct rtgenmsg *rtgenmsg = nlmsg_data(nlh);

        if (rtgenmsg->rtgen_family != AF_UNSPEC) {
            NL_SET_ERR_MSG(extack, "Invalid address family");
            return -EINVAL;
        }
    } else {
        err = nlmsg_parse_deprecated(nlh, sizeof(struct ifinfomsg),
                         tb, IFLA_MAX, ifla_policy, extack);
        if (err < 0)
            return err;

        if (tb[IFLA_TARGET_NETNSID]) {
            netnsid = nla_get_s32(tb[IFLA_TARGET_NETNSID]);
            rcu_read_lock();
            net = rtnl_get_net_ns_capable(NETLINK_CB(cb->skb).sk, netnsid);
            rcu_read_unlock();
            if (IS_ERR(net)) {
                NL_SET_ERR_MSG(extack, "Invalid target network namespace id");
                return PTR_ERR(net);
            }
        }

        if (tb[IFLA_EXT_MASK])
            filter_mask = nla_get_u32(tb[IFLA_EXT_MASK]);

        if (tb[IFLA_MASTER])
            master_idx = nla_get_u32(tb[IFLA_MASTER]);

        extfilt = tb[IFLA_EXT_FILTER_MASK];
    }

    for_each_netdev_dump(net, dev, ctx->ifindex) {
        if (master_idx && (dev->master ?
                  dev->master->ifindex != master_idx :
                  netdev_master_upper_dev_get(dev) ?
                  netdev_master_upper_dev_get(dev)->ifindex != master_idx : true))
            continue;
        if (netdev_unregistering(dev))
            continue;
        err = rtnl_fill_ifinfo(skb, dev, net,
                      RTM_NEWLINK,
                      portid, seq, 0,
                      NLM_F_MULTI,
                      filter_mask, NULL, 0, NULL,
                      rtnl_get_event(0), extfilt, 0,
                      NULL, 0, netnsid,
                      GFP_KERNEL);

        if (err < 0) {
            if (likely(skb->len))
                goto out;

            goto out_err;
        }
    }
out:
    err = skb->len;
out_err:
    if (netnsid >= 0)
        put_net(net);

    return err;
}
```

**Link Management Features**:
- **Dynamic Link Creation**: Supports creating various types of network links
- **Module Integration**: Automatic loading of link-specific kernel modules
- **Network Namespace Support**: Multi-namespace link operations
- **Attribute Parsing**: Comprehensive parsing of netlink attributes
- **Type-Specific Operations**: Support for device-type-specific operations
- **Error Reporting**: Detailed error reporting with extended acknowledgments
- **Compatibility Layer**: Backward compatibility with older userspace tools

### 4. Bridge Operations Infrastructure

**Bridge Link Configuration** - Lines 5471-5549:
```c
static int rtnl_bridge_setlink(struct sk_buff *skb, struct nlmsghdr *nlh,
                  struct netlink_ext_ack *extack)
{
    struct net *net = sock_net(skb->sk);
    struct ifinfomsg *ifm;
    struct net_device *dev;
    struct nlattr *br_spec, *attr = NULL;
    int rem, err = -EOPNOTSUPP;
    u16 flags = 0;
    bool have_flags = false;

    if (nlmsg_len(nlh) < sizeof(*ifm))
        return -EINVAL;

    ifm = nlmsg_data(nlh);
    if (ifm->ifi_family != AF_BRIDGE)
        return -EPFNOSUPPORT;

    dev = __dev_get_by_index(net, ifm->ifi_index);
    if (!dev) {
        NL_SET_ERR_MSG(extack, "Unknown ifindex");
        return -ENODEV;
    }

    br_spec = nlmsg_find_attr(nlh, sizeof(struct ifinfomsg), IFLA_AF_SPEC);
    if (br_spec) {
        nla_for_each_nested(attr, br_spec, rem) {
            if (nla_type(attr) == IFLA_BRIDGE_FLAGS) {
                if (nla_len(attr) < sizeof(flags))
                    return -EINVAL;

                have_flags = true;
                flags = nla_get_u16(attr);
                break;
            }
        }
    }

    if (!flags || (flags & BRIDGE_FLAGS_MASTER)) {
        struct net_device *br_dev = netdev_master_upper_dev_get(dev);

        if (!br_dev || !br_dev->netdev_ops->ndo_bridge_setlink) {
            err = -EOPNOTSUPP;
            goto out;
        }

        err = br_dev->netdev_ops->ndo_bridge_setlink(dev, nlh, flags,
                                 extack);
        if (err)
            goto out;

        flags &= ~BRIDGE_FLAGS_MASTER;
    }

    if ((flags & BRIDGE_FLAGS_SELF)) {
        if (!dev->netdev_ops->ndo_bridge_setlink) {
            err = -EOPNOTSUPP;
            goto out;
        }

        err = dev->netdev_ops->ndo_bridge_setlink(dev, nlh,
                              flags, extack);
        if (err)
            goto out;
    }

    if (have_flags)
        memcpy(nla_data(attr), &flags, sizeof(flags));
out:
    return err;
}
```

**Bridge Features**:
- **Master/Self Operations**: Support for bridge master and self operations
- **Flag Management**: Comprehensive bridge flag handling
- **Device Validation**: Proper validation of bridge-capable devices
- **Error Propagation**: Detailed error reporting for bridge operations
- **Netlink Integration**: Full integration with netlink message format

### 5. Forwarding Database (FDB) Management

**FDB Entry Addition** - Lines 4584-4653:
```c
static int rtnl_fdb_add(struct sk_buff *skb, struct nlmsghdr *nlh,
               struct netlink_ext_ack *extack)
{
    struct net *net = sock_net(skb->sk);
    struct ndmsg *ndm;
    struct nlattr *tb[NDA_MAX+1];
    struct net_device *dev;
    u8 *addr;
    u16 vid;
    int err;

    err = nlmsg_parse_deprecated(nlh, sizeof(*ndm), tb, NDA_MAX,
                     NULL, extack);
    if (err < 0)
        return err;

    ndm = nlmsg_data(nlh);
    if (ndm->ndm_ifindex == 0) {
        NL_SET_ERR_MSG(extack, "PF_BRIDGE FDB operations not supported");
        return -EINVAL;
    }

    dev = __dev_get_by_index(net, ndm->ndm_ifindex);
    if (dev == NULL) {
        NL_SET_ERR_MSG(extack, "Unknown ifindex");
        return -ENODEV;
    }

    if (!tb[NDA_LLADDR] || nla_len(tb[NDA_LLADDR]) != ETH_ALEN) {
        NL_SET_ERR_MSG(extack, "Invalid ethernet address");
        return -EINVAL;
    }

    if (dev->type != ARPHRD_ETHER) {
        NL_SET_ERR_MSG(extack, "FDB add only supported for Ethernet devices");
        return -EINVAL;
    }

    addr = nla_data(tb[NDA_LLADDR]);

    err = fdb_vid_parse(tb[NDA_VLAN], &vid, extack);
    if (err)
        return err;

    err = -EOPNOTSUPP;

    /* Support fdb on master device the net/bridge default case */
    if ((!ndm->ndm_flags || ndm->ndm_flags & NTF_MASTER) &&
        netif_is_bridge_port(dev)) {
        struct net_device *br_dev = netdev_master_upper_dev_get(dev);
        const struct net_device_ops *ops = br_dev->netdev_ops;

        err = ops->ndo_fdb_add(ndm, tb, dev, addr, vid,
                      nlh->nlmsg_flags, extack);
        if (err)
            goto out;
        else
            ndm->ndm_flags &= ~NTF_MASTER;
    }

    /* Embedded bridge, macvlan, and any other device support */
    if ((ndm->ndm_flags & NTF_SELF)) {
        if (dev->netdev_ops->ndo_fdb_add)
            err = dev->netdev_ops->ndo_fdb_add(ndm, tb, dev, addr,
                               vid, nlh->nlmsg_flags,
                               extack);
        else
            err = ndo_dflt_fdb_add(ndm, tb, dev, addr, vid,
                          nlh->nlmsg_flags);

        if (!err) {
            rtnl_fdb_notify(dev, addr, vid, RTM_NEWNEIGH,
                    ndm->ndm_state);
            ndm->ndm_flags &= ~NTF_SELF;
        }
    }
out:
    return err;
}
```

**FDB Features**:
- **Ethernet Address Validation**: Ensures proper MAC address format
- **VLAN Support**: Full VLAN ID parsing and validation
- **Bridge Integration**: Support for bridge master operations
- **Device Type Checking**: Validates device type compatibility
- **Notification System**: Generates notifications for FDB changes

### 6. Statistics Collection and Reporting

**Statistics Retrieval** - Lines 6237-6287:
```c
static int rtnl_stats_get(struct sk_buff *skb, struct nlmsghdr *nlh,
             struct netlink_ext_ack *extack)
{
    struct net *net = sock_net(skb->sk);
    struct net_device *dev = NULL;
    int idxattr = 0, prividx = 0;
    struct if_stats_msg *ifsm;
    struct sk_buff *nskb;
    u32 filter_mask;
    int err;

    if (nlmsg_len(nlh) < sizeof(*ifsm))
        return -EINVAL;

    ifsm = nlmsg_data(nlh);
    if (ifsm->pad1 || ifsm->pad2 || ifsm->ifindex < 0)
        return -EINVAL;

    if (ifsm->ifindex == 0)
        return -EINVAL;

    filter_mask = ifsm->filter_mask;
    if (!filter_mask) {
        NL_SET_ERR_MSG(extack, "Filter mask must be set for stats get");
        return -EINVAL;
    }

    dev = __dev_get_by_index(net, ifsm->ifindex);
    if (!dev)
        return -ENODEV;

    nskb = nlmsg_new(if_nlmsg_stats_size(dev, filter_mask), GFP_KERNEL);
    if (!nskb)
        return -ENOBUFS;

    err = rtnl_fill_statsinfo(nskb, dev, RTM_NEWSTATS,
                 NETLINK_CB(skb).portid, nlh->nlmsg_seq, 0,
                 0, filter_mask, &idxattr, &prividx,
                 extack);
    if (err < 0) {
        /* -EMSGSIZE implies BUG in if_nlmsg_stats_size */
        WARN_ON(err == -EMSGSIZE);
        kfree_skb(nskb);
    } else {
        err = rtnl_unicast(nskb, net, NETLINK_CB(skb).portid);
    }

    return err;
}
```

**Statistics Features**:
- **Filter Mask Support**: Selective statistics collection based on filters
- **Device Validation**: Proper device existence and state checking
- **Dynamic Sizing**: Efficient message sizing based on requested statistics
- **Error Handling**: Comprehensive error reporting and resource cleanup
- **Unicast Response**: Direct response to requesting process

## Advanced Features

### 1. Multicast Database (MDB) Management

**MDB Operations**:
- **Entry Management**: Add, delete, and modify multicast database entries
- **Group Membership**: Manage multicast group memberships
- **Bridge Integration**: Full integration with bridge multicast forwarding
- **VLAN Support**: VLAN-aware multicast database operations

### 2. Network Namespace Integration

**Namespace Features**:
- **Cross-Namespace Operations**: Support for operations across network namespaces
- **Namespace Validation**: Proper validation of namespace access permissions
- **Reference Management**: Safe reference counting for namespace objects
- **Migration Support**: Support for moving devices between namespaces

### 3. Extended Statistics and Offload Support

**Advanced Statistics**:
- **Hardware Offload Stats**: Support for hardware-offloaded statistics
- **Per-Queue Statistics**: Queue-specific statistics collection
- **Custom Statistics**: Extensible framework for device-specific statistics
- **Real-time Monitoring**: Efficient real-time statistics updates

### 4. Link Aggregation and Bonding

**Bonding Support**:
- **Slave Management**: Management of bond slave interfaces
- **Load Balancing**: Configuration of load balancing modes
- **Failover Support**: Automatic failover configuration
- **Link Monitoring**: Active link monitoring capabilities

## Performance Optimizations

### 1. RCU-Protected Operations

**Lock-Free Access**:
- **RCU Read Sections**: Minimal critical sections for read operations
- **Safe Handler Lookup**: Lock-free handler table lookups
- **Reference Counting**: Atomic reference counting for objects
- **Deferred Cleanup**: RCU-based cleanup for safe memory reclamation

### 2. Efficient Message Processing

**Processing Optimization**:
- **Batch Operations**: Support for batched netlink operations
- **Minimal Copying**: Zero-copy operations where possible
- **Fast Path**: Optimized fast paths for common operations
- **Cache-Friendly**: Data structure layout optimized for cache efficiency

### 3. Memory Management

**Efficient Memory Usage**:
- **Deferred SKB Cleanup**: SKB cleanup deferred to avoid lock contention
- **Memory Pooling**: Efficient memory pool usage for frequent allocations
- **Alignment Optimization**: Proper alignment for performance
- **Resource Tracking**: Comprehensive resource usage tracking

## Security Considerations

### 1. Capability-Based Access Control

**Permission Enforcement**:
- **CAP_NET_ADMIN**: Strict enforcement of network administration capabilities
- **Namespace Isolation**: Proper isolation between network namespaces
- **User Credential Checking**: Validation of user credentials for operations
- **Audit Integration**: Integration with kernel audit subsystem

### 2. Input Validation

**Security Validation**:
- **Message Length Checking**: Comprehensive validation of message lengths
- **Attribute Validation**: Thorough validation of all netlink attributes
- **Range Checking**: Proper bounds checking on all numeric inputs
- **Buffer Overflow Protection**: Protection against buffer overflow attacks

### 3. Resource Management Security

**Resource Protection**:
- **Memory Limits**: Enforcement of memory usage limits
- **Rate Limiting**: Protection against DoS through rate limiting
- **Reference Counting**: Proper reference counting to prevent use-after-free
- **Module Security**: Safe loading and unloading of kernel modules

### 4. Network Security Integration

**Security Framework Integration**:
- **LSM Integration**: Integration with Linux Security Modules
- **Netfilter Hooks**: Support for netfilter security hooks
- **Encryption Support**: Framework for network encryption configuration
- **Access Control Lists**: Support for network access control

The routing netlink interface represents the cornerstone of Linux network administration, providing a comprehensive, secure, and efficient interface for managing all aspects of the kernel's networking subsystem. Its sophisticated design enables everything from simple interface configuration to complex network topology management, making it the foundation upon which all modern Linux network management tools are built.