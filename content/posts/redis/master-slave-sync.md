---
title: redis 主从同步
categories:
  - redis
slug: redis-master-slave-sync
output:
  blogdown::html_page:
    toc: true
---
# 为什么需要主从同步

从节点通过主从同步和主节点保持相同的数据，这样一个分片可以做到
1. 故障转移（failover），当主节点宕机时，从节点可以被提升成主节点
2. 负载均衡，从节点可以承载部分的读流量。
# redis 中主从同步的演变
- 2.8 版本之前 redis 仅支持全量同步sync
- 2.8 ～ 4.0 新增了 psync 命令，采用全量同步和增量同步结合的方式，并且在主从同步断联时通过offset信息继续通过增量同步方式
- 4.0 版本之后优化了 psync 方式，称之为psync2，解决了psync在故障转移后无法支持增量同步的问题
# 主从同步流程
## 1. 建立连接阶段
- 从节点执行复制命令
```
redis-cli cluster replicate ip port
```
- 如果主节点设置了密码，则需要鉴权
主节点密码设置
```
requirepass masterpassword
```
从节点密码设置
```
masterauth slavepassword
```

```C
/* AUTH with the master if required. */
if (server.masterauth) {
	char *args[3] = {"AUTH",NULL,NULL};
	size_t lens[3] = {4,0,0};
	int argc = 1;
	if (server.masteruser) {
		args[argc] = server.masteruser;
		lens[argc] = strlen(server.masteruser);
		argc++;
	}
	args[argc] = server.masterauth;
	lens[argc] = sdslen(server.masterauth);
	argc++;
	err = sendCommandArgv(conn, argc, args, lens);
	if (err) goto write_error;
}
```
- 协商协议，从节点发送复制协议版本
```c++
err = sendCommand(conn,"REPLCONF",
                          "capa","eof","capa","psync2",
                          server.repl_rdb_channel ? "capa" : NULL, "rdb-channel-repl", NULL);
```
## 2. 从节点发送PSYNC
```c
/* Issue the PSYNC command, if this is a master with a failover in
 * progress then send the failover argument to the replica to cause it
 * to become a master */
if (server.failover_state == FAILOVER_IN_PROGRESS) {
	reply = sendCommand(conn,"PSYNC",psync_replid,psync_offset,"FAILOVER",NULL);
} else {
	reply = sendCommand(conn,"PSYNC",psync_replid,psync_offset,NULL);
}
```
## 3. 主节点收到 PSYNC 之后，判断是否可进行增量同步
- 条件 1： 
主节点维护两个复制ID：`replid`（当前主ID）和 `replid2`（前主ID）当从节点提供的ID既不等于当前主ID，也不等于前主ID（或使用前主ID但偏移超出历史范围时）**进行全量同步**
```c++
    /* Is the replication ID of this master the same advertised by the wannabe
     * slave via PSYNC? If the replication ID changed this master has a
     * different replication history, and there is no way to continue.
     *
     * Note that there are two potentially valid replication IDs: the ID1
     * and the ID2. The ID2 however is only valid up to a specific offset. */
    if (strcasecmp(master_replid, server.replid) &&
        (strcasecmp(master_replid, server.replid2) ||
         psync_offset > server.second_replid_offset))
         {
	        .....
	        goto need_full_resync;
         }
`````
- 条件 2
复制积压缓冲区检查：积压缓冲区未初始化 ；请求偏移小于缓冲区起始偏移（数据已被覆盖）；请求偏移超过缓冲区最大偏移（数据尚未产生），**进行全量同步**
```c
/* We still have the data our slave is asking for? */
if (!server.repl_backlog ||
	psync_offset < server.repl_backlog->offset ||
	psync_offset > (server.repl_backlog->offset + server.repl_backlog->histlen))
{
	serverLog(LL_NOTICE,
		"Unable to partial resync with replica %s for lack of backlog (Replica request was: %lld).", replicationGetSlaveName(c), psync_offset);
	if (psync_offset > server.master_repl_offset) {
		serverLog(LL_WARNING,
			"Warning: replica %s tried to PSYNC with an offset that is greater than the master replication offset.", replicationGetSlaveName(c));
	}
	goto need_full_resync;
}
```
## 4. 全量同步
主节点发送`+RDBCHANNELSYNC <client-id>` 通知从节点建立专用传输通道，从节点接收到此请求将主节点的通道ID与自身关联，形成双向绑定关系。
主节点通过bgsave 进程生成 rdb，在写入 rdb过程中将增量请求写入repl_backlog中
```c
/* Replication backlog is not a separate memory, it just is one consumer of
 * the global replication buffer. This structure records the reference of
 * replication buffers. Since the replication buffer block list may be very long,
 * it would cost much time to search replication offset on partial resync, so
 * we use one rax tree to index some blocks every REPL_BACKLOG_INDEX_PER_BLOCKS
 * to make searching offset from replication buffer blocks list faster. */
typedef struct replBacklog {
    listNode *ref_repl_buf_node; /* Referenced node of replication buffer blocks,
                                  * see the definition of replBufBlock. */
    size_t unindexed_count;      /* The count from last creating index block. */
    rax *blocks_index;           /* The index of recorded blocks of replication
                                  * buffer for quickly searching replication
                                  * offset on partial resynchronization. */
    long long histlen;           /* Backlog actual data length */
    long long offset;            /* Replication "master offset" of first
                                  * byte in the replication backlog buffer.*/
} replBacklog;
```
当 bgsave 完成后，redis 会调用updateSlavesWaitingBgsave，注册写事件sendBulkToSlave，将 rdb 传输给从节点，从节点通过readSyncBulkPayload函数读取 rdb 流，从节点读取完成后发送REPLCONF ACK+offset，主节点通过replconfCommand确认 offset，之后进入增量同步模式
## 5. 增量同步

```c
 /*
 * * Replica state machine *
 *
 * Main channel state
 * ┌───────────────────┐
 * │RECEIVE_PING_REPLY │
 * └────────┬──────────┘
 *          │ +PONG
 * ┌────────▼──────────┐
 * │SEND_HANDSHAKE     │                     RDB channel state
 * └────────┬──────────┘            ┌───────────────────────────────┐
 *          │+OK                ┌───► RDB_CH_SEND_HANDSHAKE         │
 * ┌────────▼──────────┐        │   └──────────────┬────────────────┘
 * │RECEIVE_AUTH_REPLY │        │    REPLCONF main-ch-client-id <clientid>
 * └────────┬──────────┘        │   ┌──────────────▼────────────────┐
 *          │+OK                │   │ RDB_CH_RECEIVE_AUTH_REPLY     │
 * ┌────────▼──────────┐        │   └──────────────┬────────────────┘
 * │RECEIVE_PORT_REPLY │        │                  │ +OK
 * └────────┬──────────┘        │   ┌──────────────▼────────────────┐
 *          │+OK                │   │  RDB_CH_RECEIVE_REPLCONF_REPLY│
 * ┌────────▼──────────┐        │   └──────────────┬────────────────┘
 * │RECEIVE_IP_REPLY   │        │                  │ +OK
 * └────────┬──────────┘        │   ┌──────────────▼────────────────┐
 *          │+OK                │   │ RDB_CH_RECEIVE_FULLRESYNC     │
 * ┌────────▼──────────┐        │   └──────────────┬────────────────┘
 * │RECEIVE_CAPA_REPLY │        │                  │+FULLRESYNC
 * └────────┬──────────┘        │                  │Rdb delivery
 *          │                   │   ┌──────────────▼────────────────┐
 * ┌────────▼──────────┐        │   │ RDB_CH_RDB_LOADING            │
 * │SEND_PSYNC         │        │   └──────────────┬────────────────┘
 * └─┬─────────────────┘        │                  │ Done loading
 *   │PSYNC (use cached-master) │                  │
 * ┌─▼─────────────────┐        │                  │
 * │RECEIVE_PSYNC_REPLY│        │    ┌────────────►│ Replica streams replication
 * └─┬─────────────────┘        │    │             │ buffer into memory
 *   │                          │    │             │
 *   │+RDBCHANNELSYNC client-id │    │             │
 *   ├──────┬───────────────────┘    │             │
 *   │      │ Main channel           │             │
 *   │      │ accumulates repl data  │             │
 *   │   ┌──▼────────────────┐       │     ┌───────▼───────────┐
 *   │   │ REPL_TRANSFER     ├───────┘     │    CONNECTED      │
 *   │   └───────────────────┘             └────▲───▲──────────┘
 *   │                                          │   │
 *   │                                          │   │
 *   │  +FULLRESYNC    ┌───────────────────┐    │   │
 *   ├────────────────► REPL_TRANSFER      ├────┘   │
 *   │                 └───────────────────┘        │
 *   │  +CONTINUE                                   │
 *   └──────────────────────────────────────────────┘
 */
```
