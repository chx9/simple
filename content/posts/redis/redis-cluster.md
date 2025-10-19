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

分布式系统往往通过一主多从模式对外提供服务，保证服务的容灾，然而当数据量不断扩大，往往会遇到**横向扩展**的问题，需要通过多主多从的方式存储数据，然而当数据存储在多个节点上，客户端应该如何路由并获取数据？业界常常采用的策略有几种:

1. 使用 proxy来做路由，proxy 接受到客户端发来的请求，根据集群的路由规则转发到 redis 节点，等到其回复，收到回复之后由将请求回复给客户端。这样的场景简化了客户端实现逻辑，可以把集群当做一个单例。

![支持redis节点高可用的twemproxy-社区博客-网易数帆](https://nos.netease.com/cloud-website-bucket/2018070312245220eb4dca-ea94-4529-b569-8468bb7c474d.png)

2. 实现复杂的集群模式客户端，客户端根据维护的路由规则访问不同的节点。

# 集群模式简介

redis 从 3.0 版本起支持集群模式，在该模式下，数据被分散存储到多个slot 上，每个节点管理一些 slots，并且每个节点都维护了集群的拓扑结构，包括主从关系、slot 归属等。

**redis集群模式的目标：**

- **高性能、可线性增长** :不使用 proxy（增加一跳）、异步复制，不对 value 执行合并操作，最多可以扩展到 1000 个节点。
- **写入安全**：通过多个主节点和主从同步保证写入的安全性。
- **高可用**：一个主节点往往会挂有 1 个到多个从节点，当主节点宕机时从节点会被选举成为主节点。当主节点下从节点全部不可用时，redis 的副本漂移机制会从有多个从节点的主节点中拿到一个从节点，保证数据的安全性。

![Redis cluster and sentinel with docker-From Zero to Hero -part III | by  ChickenBenny | DevOps.dev](https://miro.medium.com/v2/resize:fit:1400/1*L7OkvJ5U-IWQmeDUe1t6wg.png)

上图中就是一个规格为 3 主 6 从的redis 集群，master A 管理 0 到 5460 的 slots，master B 管理 5461 到 10922 的 slots，master C 管理 10923 到 16383 的节点。一个主节点和其从节点组成一个分片，在图中 Master A 和 Replica A1、Replica A2 为一个分片，Replica A1 和 A2 异步复制 master A 的数据确保高可用。

# 集群模式实现细节

## slot哈希槽

### slot计算

集群模式下，数据被分配到各个哈希槽，简称 slot，redis 集群维护了 16384 个 slots。在集群模式下，当客户端想发起一个写入请求时，首先需要知道应该向哪个主节点发起该请求。客户端通过 CRC16哈希算法，算出 key 对应的 slot

``````c
SLOT = CRC16(key) % 16384
``````

计算出slot 后，根据集群的slot 信息，可以知道哪个主节点维护了该slot，然后将请求发送至该主节点，完成一个写请求。 

在 redis 集群中，写请求只能发送给主节点，如果发送给从节点，从节点会返回错误，这是因为从节点只是主节点的副本。

redis 额外提供了一种特殊的方式可以保证 2 个 key 总是可以哈希到同一个 slot 中，就是当 key 中出现大括号{}对时，只计算大括号中的，例如 key1{abc} 和 key2{abc}的 slot 是相同的，因为在进行哈希计算时，只计算{abc}的哈希

### slot在集群中的表示

在 redis 集群模式下，总共有 2^14 =16384 个slot，每个节点维护部分 slot，在 redis 中，节点数据结构是`clusterNode`，表示了一个节点。该数据结构维护了一个 unsigned char 的数组，表示该节点所维护的 slots，下面代码中 CLUSTER_SLOT 的值为 16384，之所以除以 8 是因为unsigned char 为一个字节，也即 8位。

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

### slot迁移

当集群需要扩容时原有分片需要将部分 slot 迁移到新的分片上，当集群需要缩容时被缩掉的分片需要将slot 迁移到其余分片上，因此 slot 迁移也是一个重要、频繁的运维操作。

redis 迁移过程分为几步

1. 在迁出节点将被迁移的 slot 标记为 migrating

``````
set slot migrating
``````

1. 将
