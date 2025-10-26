---
title: redis 集群模式
categories:
  - redis
slug: redis-cluster
output:
  blogdown::html_page:
    toc: true
---

# 背景

分布式系统往往通过一主多从模式对外提供服务，保证服务的容灾，然而当数据量不断扩大，往往会遇到**横向扩展**的问题，需要通过多主多从的方式存储数据，然而当数据存储在多个节点上，客户端应该如何路由请求？业界常常采用的策略有 2种:

1. 使用 proxy来做路由，proxy 接受到客户端发来的请求，根据集群的路由规则转发到 redis 节点，等到其回复，收到回复之后由将请求回复给客户端。这样的场景简化了客户端实现逻辑，**可以把集群当做一个单例**。

2. 实现复杂的集群模式客户端，客户端根据维护的路由规则访问不同的节点。

然而以 proxy 接入的方式需要增加额外的一跳；如果把路由交给客户端去维护，那客户端实现将会变得非常复杂，不便于管理。

# 集群模式简介

redis 从 3.0 版本起支持集群模式，在该模式下，数据被分散存储到多个slot 上，每个节点管理一些 slots，并且每个节点都维护了集群的拓扑结构，包括主从关系、slot 归属等。

**redis集群模式的目标：**

- **高性能**:不使用 proxy（增加一跳）、异步复制。
- **高可用**：一个主节点往往会挂有 1 个到多个从节点，当主节点宕机时从节点会被选举成为主节点。当主节点下从节点全部不可用时，redis 的副本漂移机制会从有多个从节点的主节点中拿到一个从节点，保证数据的安全性。
- 可扩展：可增加分片和从节点数量，最多可以扩展到 1000 个节点。

![Redis cluster and sentinel with docker-From Zero to Hero -part III | by  ChickenBenny | DevOps.dev](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/3657e0c925b04f069ee3922f0682da83~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgaW1jaHg5:q75.awebp?rk3s=f64ab15b&x-expires=1762094684&x-signature=s2J763TwT32YXTkHk%2BsT3oJdsO8%3D)

上图中就是一个规格为 3 主 6 从的redis 集群，一个主节点和其从节点组成一个分片，在图中 Master A 和 Replica A1、Replica A2 为一个分片，Replica A1 和 A2 异步复制 master A 的数据确保高可用；master A、B、C 分别管理0-5460、5461 - 10922 、 10923 - 16383 的 slots。

# 集群模式实现细节

## slot哈希槽

### slot计算

集群模式下，数据被分配到各个哈希槽，简称 slot，redis 集群维护了 16384 个 slots。在集群模式下，当客户端想发起一个写入请求时，首先需要知道应该向哪个主节点发起该请求。客户端通过 CRC16哈希算法，算出 key 对应的 slot

``````c
SLOT = CRC16(key) % 16384
``````

计算出slot 后，根据集群的slot 信息，可以知道哪个主节点维护了该slot，然后将请求发送至该主节点，完成一个写请求。 

redis 额外提供了一种特殊的方式可以保证 2 个 key 总是可以哈希到同一个 slot 中，就是当 key 中出现大括号{}对时，只计算大括号中的，例如 key1{abc} 和 key2{abc}的 slot 是相同的，因为在进行哈希计算时，只计算{abc}的哈希

在 redis 集群中，写请求只能发送给主节点，如果发送给从节点，从节点会返回 MOVED 错误，这是因为从节点只是主节点的副本，因此只接受来自主节点的写请求。

### slot在集群中的表示

在 redis 集群模式下，总共有 2^14 =16384 个slot，每个节点维护部分 slot，在 redis 中，节点数据结构是`clusterNode`，表示了一个节点。该数据结构维护了一个数组，表示该节点所维护的 slots，下面代码中 CLUSTER_SLOT 的值为 16384，之所以除以 8 是因为unsigned char 为一个字节，也即 8位，每一位都表示了一个 slot。

``````c
struct _clusterNode {
    unsigned char slots[CLUSTER_SLOTS/8]; /* slots handled by this node */
}
``````

除此之外还有`clusterState` 数据结构，该数据结构维护的是整个集群的状态信息

`````c
struct clusterState {
    clusterNode *myself;  /* This node */
    uint64_t currentEpoch;
    int state;            /* CLUSTER_OK, CLUSTER_FAIL, ... */
    int size;             /* Num of master nodes with at least one slot */
    dict *nodes;          /* Hash table of name -> clusterNode structures */
    dict *shards;         /* Hash table of shard_id -> list (of nodes) structures */
    dict *nodes_black_list; /* Nodes we don't re-add for a few seconds. */
    clusterNode *migrating_slots_to[CLUSTER_SLOTS];
    clusterNode *importing_slots_from[CLUSTER_SLOTS];
    clusterNode *slots[CLUSTER_SLOTS];
    // ...
}
`````

其中`clusterNode *slots[CLUSTER_SLOTS];`代表了 slot 到节点的映射。

currentEpoch 表示了当前的配置纪元，它是单调的增的，用于维护集群内状态的一致性，确保每个节点有相同的共识。

## 集群的创建

### 启动单节点

如果要创建集群，那么在节点启动时要将配置文件中的cluster enable 的配置为 yes，否则无法开启。

启动 redis 单例后发送任意读写请求，会发现返回`CLUSTERDOWN The cluster is down`的 error，原因是当前 slot 并未分配，所以集群状态为 failed

### 节点相互meet

节点之间互相认识对方可以通过两种方式

1. 相互发送 cluster meet 命令。`cluster MEET <ip> <port> `
2. 集群节点之间不断传播的 gossip 消息，消息中带有节点维护的集群拓扑信息。如 AB 之间互相 meet，C 又和 A meet，那么此时 B 传递给 A 的 gossip 消息包含 C 的信息，因此 A 也知道了 C 的存在。

通过 cluster meet 消息可以跟另一个节点建立连接 ，注意，redis 集群间通信的端口为工作端口+10000，如果工作端口为 6379，那么集群通信端口为16379

cluster meet 的过程

1. 客户端向 A 节点发送 `[cluster meet ip_B port_B]`请求，A 接收到请求后新建一个节点 B，并加入到`cluster->nodes`中，并将其标记为`CLUSTER_NODE_HANDSHAKE|CLUSTER_NODE_MEET`。
2. 在 A 的 `clusterCron` 中向 B节点建立连接，由于B 节点被 A 标记为`CLUSTER_NODE_MEET`，因此 A 节点建立连接后向 B 发送MEET 消息，B 接收到 MEET 消息后，将 A 加入`cluster->nodes`中，并发送 pong 给 A 节点
3. A节点接收到 B 发来的 Pong 消息，A 节点认为已经完成握手。
4. B在后续的`clusterCron`也会向 A 建立连接，互相发送 ping pong，从而完成握手。

### gossip协议

redis 除了和客户端通信的 RESP 协议以外，还定义了一个集群间通信的 gossip 协议，该协议使用集群通信端口，只用于节点和节点之间的通信。

![Understanding the Failover Mechanism of Redis Cluster | by Alibaba Cloud |  Medium](https://p9-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/1c136c5da6df4e26947971ca62463a95~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgaW1jaHg5:q75.awebp?rk3s=f64ab15b&x-expires=1762094684&x-signature=xTOfEbaOGiGnQEH0g%2B7rMp16ldo%3D)

gossip 协议把网络上所有节点都视为平等而普通的一员，没有中心化节点或者主节点的概念。这些特点使得其具有极强的鲁棒性和可扩展性

节点之间定期向其他节点发送 gossip 信息，消息体包含了集群当前维护的拓扑信息，gossip 信息主要有以下几种

- **CLUSTERMSG_TYPE_PING**
1. 在 clusterCron 中每隔一秒，节点就会随机从5 个节点中选出一个最早接收到对方发来pong 的节点，向其发送 ping
2. 从节点手动 failover【切主】 master 需要向从节点 发送 ping 消息，该消息中包含了`server.cluster->mf_master_offset`（即主节点当前的复制偏移量）从节点收到后知道主节点已经暂停写操作到这个 offset，于是可以安全地完成主从切换。
3. 在 clusterCron 中假设收到某个节点发送的 pong 消息时间已经超过了 cluster-node-timeout 的一半，向其发送 ping

- **CLUSTERMSG_TYPE_PONG**

 1. 收到 ping 或 meet 消息，回复 pong
 2. 从节点发生主从切换后发送 pong 消息给所有其他节点让他们知道我切换成了主节点
 3. import slot 之后让其他节点知道这些 slot 归我所属

- **CLUSTERMSG_TYPE_MEET**

​	   请求节点相互认识

- **CLUSTERMSG_TYPE_FAIL**

​      某个节点被标记为 FAIL，向其他所有节点发送 fail 消息

- **CLUSTERMSG_TYPE_PUBLISH**

​       用于`publish`命令，publish 的消息将会在整个集群中传播

#### gossip消息体

一条 gossip 消息主要由`clusterMsg`和`clusterMsgData`组成

先看`clusterMsg`的结构

```c
typedef struct {
    char sig[4];        /* Signature "RCmb" (Redis Cluster message bus). */
    uint32_t totlen;    /* Total length of this message */
    uint16_t ver;       /* Protocol version, currently set to 1. */
    uint16_t port;      /* Primary port number (TCP or TLS). */
    uint16_t type;      /* Message type */
    uint16_t count;     /* Only used for some kind of messages. */
    uint64_t currentEpoch;  /* The epoch accordingly to the sending node. */
    uint64_t configEpoch;   /* The config epoch if it's a master, or the last
                               epoch advertised by its master if it is a
                               slave. */
    uint64_t offset;    /* Master replication offset if node is a master or
                           processed replication offset if node is a slave. */
    char sender[CLUSTER_NAMELEN]; /* Name of the sender node */
    unsigned char myslots[CLUSTER_SLOTS/8];
    char slaveof[CLUSTER_NAMELEN];
    char myip[NET_IP_STR_LEN];    /* Sender IP, if not all zeroed. */
    uint16_t extensions; /* Number of extensions sent along with this packet. */
    char notused1[30];   /* 30 bytes reserved for future usage. */
    uint16_t pport;      /* Secondary port number: if primary port is TCP port, this is 
                            TLS port, and if primary port is TLS port, this is TCP port.*/
    uint16_t cport;      /* Sender TCP cluster bus port */
    uint16_t flags;      /* Sender node flags */
    unsigned char state; /* Cluster state from the POV of the sender */
    unsigned char mflags[3]; /* Message flags: CLUSTERMSG_FLAG[012]_... */
    union clusterMsgData data;
} clusterMsg;
```

#### currentEpoch和configEpoch

|              | 介绍 | 作用 |
| -- | ------- | ------------ |
| currentEpoch | 集群共识的 epoch，每个节点都维护了集群的 currentEpoch ，在节点互相通信过程中如何发现某个节点发来的 currentEpoch 大于本节点维护的，就会更新至发送者的 currentEpoch，最终所有节点的 currentEpoch 都是相同的 | 集群的配置共识。当某个节点发生配置变更，如 slot 归属变更，就会增加增加 currentEpoch 值，其他节点发现该节点的 currentEpoch 最大，便更新配置和 currentEpoch 值 |
| configEpoch  | 集群每个**主节点**都维护了一个 unique 的 configEpoch，从节点则继承主节点的 configEpoch | 主要是解决配置冲突。当两个节点声都声称是某个slot 的 owner，取 configEpoch 最高的。当节点的配置（如角色、槽位归属）发生变更时，节点会将 configEpoch 复制成 currentEpoch+1 以获得权威性 |

每个节点都维护了一个自身的 currentEpoch 和集群整体的 currentEpoch（还有选举的 epoch：failover_auth_epoch，另外存储了其他节点的 configEpoch 的信息（存储在 clusterNode 结构中）

通过`cluster nodes`命令可以查看到记录的所有节点的 configEpoch

![image.png](https://p3-xtjj-sign.byteimg.com/tos-cn-i-73owjymdk6/edfe9f4ff0764bbaa3c0c2e01a1569a1~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgaW1jaHg5:q75.awebp?rk3s=f64ab15b&x-expires=1761893415&x-signature=BVQH1X1LeMcBY0SnL40C%2FpI7yQo%3D)

手动变更 （无需集群共识）currentEpoch 的几种方法

- cluster set-config-epoch <epoch>
手动设置 configEpoch，一般在故障情况下人为介入，例如网络分割恢复后调大某一端的 configEpoch让其有更大的优先级。
- cluster bumpepoch
调用clusterBumpConfigEpochWithoutConsensus，只有节点不是集群中最大的configEpoch 或者configEpoch 为 0 时，改命令把currentEpoch+1 后将 configEpoch 设置为该值，作用和 cluster set-config-epoch 类似
- cluster setslot <slotid> node <nodeid>
节点设置 slot 归属时触发 clusterBumpConfigEpochWithoutConsensus
- takeover 模式的 manual failover

集群共识自动变更

- 解决 configEpoch 冲突
当前 master 收到其他master 发来的 gossip 信息，发现其 configEpoch 和自己相同，此时需要通过clusterHandleConfigEpochCollision将本地 configEpoch+1 解决冲突
- slave 发起 failover 选举
slave 发现 master 故障或者手动发起 failover，slave 将自身currentEpoch+1 后赋值给failover_auth_epoch发起选举。选举成功后将currentEpoch 更新至 failover_auth_epoch

#### clusterMsgData

clusterMsgData 是 clusterMsg 的主体消息，它是一个 union 结构，可以表示多种 ping/pong、fail、publish、update、module 等多种消息结构

```c
union clusterMsgData {
    /* PING, MEET and PONG */
    struct {
        /* Array of N clusterMsgDataGossip structures */
        clusterMsgDataGossip gossip[1];
        /* Extension data that can optionally be sent for ping/meet/pong
         * messages. We can't explicitly define them here though, since
         * the gossip array isn't the real length of the gossip data. */
    } ping;

    /* FAIL */
    struct {
        clusterMsgDataFail about;
    } fail;

    /* PUBLISH */
    struct {
        clusterMsgDataPublish msg;
    } publish;

    /* UPDATE */
    struct {
        clusterMsgDataUpdate nodecfg;
    } update;

    /* MODULE */
    struct {
        clusterMsgModule msg;
    } module;
};

```

ping/pong 消息是一个可变的数组，数据结构是clusterMsgDataGossip，对于集群中的节点，重要信息有：

- ping_sent：什么时候向该节点发送过 ping
- poing_recieved：什么时候收到该节点的 pong

这两个信息对于判断节点是否健康有重要意义

```c
typedef struct {
    char nodename[CLUSTER_NAMELEN];
    uint32_t ping_sent;
    uint32_t pong_received;
    char ip[NET_IP_STR_LEN];  /* IP address last time it was seen */
    uint16_t port;              /* primary port last time it was seen */
    uint16_t cport;             /* cluster port last time it was seen */
    uint16_t flags;             /* node->flags copy */
    uint16_t pport;             /* secondary port last time it was seen */
    uint16_t notused1;
} clusterMsgDataGossip;
```

fail 消息的主体结构是clusterMsgDataFail，它很简单，只包含了 fail节点的名字

```c
typedef struct {
    char nodename[CLUSTER_NAMELEN];
} clusterMsgDataFail;
```

publich 消息主体结构为clusterMsgDataPublish，包含了 channel 和 message

```c
typedef struct {
    uint32_t channel_len;
    uint32_t message_len;
    unsigned char bulk_data[8]; /* 8 bytes just as placeholder. */
} clusterMsgDataPublish;
```

update 消息的主体结构为clusterMsgDataUpdate,包含了某个节点的 configEpoch以及它负责的 slots。当集群 slot 归属信息发生变化时，通过 update 消息通知其他节点

```c
typedef struct {
    uint64_t configEpoch; /* Config epoch of the specified instance. */
    char nodename[CLUSTER_NAMELEN]; /* Name of the slots owner. */
    unsigned char slots[CLUSTER_SLOTS/8]; /* Slots bitmap. */
} clusterMsgDataUpdate;
```

#### 集群failover

集群 failover 是指集群主节点发生故障不可用，将从节点提升为主节点。

##### 过程

一下介绍一个完整的 manual failover，可以通过向某 slave 节点发送 cluster failover触发。

##### 主从协商

1. slave 向其 master 发送CLUSTERMSG_TYPE_MFSTART 请求
2. master 收到请求



failover 可以是手动，通过向从节点发送 `cluster failover`命令可以手动启动 failover，改命令有额外参数如下

- cluster failover force：正常情况下从节点在 failover 前需要和当前**主节点数据偏移量保持一致**，force 直接跳过这一步。
- cluster failover takeover：更加暴力形式，完全不需要和主节点进行通信，直接接管。

this is a update