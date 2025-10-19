---
title: redis 持久化
categories:
  - redis
slug: redis-persistence
output:
  blogdown::html_page:
    toc: true
---

# 为什么 redis 需要持久化
redis 作为内存数据库，数据的存储是放在内存中的，内存中的数据一旦断电或者重启就会丢失，因此需要持久化。在`loadDataFromDisk`函数中redis 会从aof 或 rdb 中重新加载数据, 由于 AOF 往往数据更加全一些，因此redis 会优先选择 AOF 作为重建数据的方式。
```c
void loadDataFromDisk(void) {
    long long start = ustime();
    if (server.aof_state == AOF_ON) {
        int ret = loadAppendOnlyFiles(server.aof_manifest);
        if (ret == AOF_FAILED || ret == AOF_OPEN_ERR)
            exit(1);
        if (ret != AOF_NOT_EXIST)
            serverLog(LL_NOTICE, "DB loaded from append only file: %.3f seconds", (float)(ustime()-start)/1000000);
        updateReplOffsetAndResetEndOffset();
    } else {
       // load rdb 
```
# RDB
RDB是Redis DataBase 的缩写，Redis 默认的持久化方式。RDB 持久化是将数据以**快照**的形式保存到磁盘上。 可以主动通过 save 或者 bgsave 命令来触发 RDB 持久化。
## SAVE命令
SAVE命令会阻塞当前redis服务器，直到持久化完成。当然如果当前已经有线程在执行持久化操作，那么SAVE命令会直接返回失败。
```c
void saveCommand(client *c) {
    if (server.child_type == CHILD_TYPE_RDB) {
        addReplyError(c,"Background save already in progress");
        return;
    }

    server.stat_rdb_saves++;

    rdbSaveInfo rsi, *rsiptr;
    rsiptr = rdbPopulateSaveInfo(&rsi);
    if (rdbSave(SLAVE_REQ_NONE,server.rdb_filename,rsiptr,RDBFLAGS_NONE) == C_OK) {
        addReply(c,shared.ok);
    } else {
        addReplyErrorObject(c,shared.err);
    }
}
```
## BGSAVE命令
BGSAVE 命令会创建一个子进程来执行持久化操作，父进程会阻塞直到持久化完成。

```c
    // bgsaveCommand
    if (server.child_type == CHILD_TYPE_RDB) {
        addReplyError(c,"Background save already in progress");
    } else if (hasActiveChildProcess() || server.in_exec) {
        if (schedule || server.in_exec) {
            server.rdb_bgsave_scheduled = 1;
            addReplyStatus(c,"Background saving scheduled");
        } else {
            addReplyError(c,
            "Another child process is active (AOF?): can't BGSAVE right now. "
            "Use BGSAVE SCHEDULE in order to schedule a BGSAVE whenever "
            "possible.");
        }
    } else if (rdbSaveBackground(SLAVE_REQ_NONE,server.rdb_filename,rsiptr,RDBFLAGS_NONE) == C_OK) {
        addReplyStatus(c,"Background saving started");
    } else {
```
注意到上面代码中，如果当前存在其他子进程（如 AOF 重写），那么 BGSAVE 命令会直接返回失败。如果希望在其他子进程执行完成后自动执行 RDB 备份，而不是立即失败，可以使用 BGSAVE SCHEDULE 命令。该命令会将 RDB 备份请求加入调度队列，当前面的子进程任务完成后，在 serverCron 中Redis 会自动执行被调度的 RDB 备份任务。
## RDB配置
RDB 有以下几个配置项：

1. `stop-writes-on-bgsave-error`
当RDB 持久化发生错误时，是否停止写入操作。默认值为 yes，表示发生错误时停止写入操作。
2. `rdbcompression`
是否对 RDB 文件进行压缩。默认值为 yes，表示对 RDB 文件进行压缩, 默认 redis 使用 LZF 算法
3. `rdbchecksum`
是否对 RDB 文件进行校验。默认值为 yes，表示对 RDB 文件进行校验。 
4. save point <seconds> <changes>
配置保存点(save point)，Redis 如果每 N 秒数据发生了 M 次改变就保存快照文件
``` bash
# 这个保存点配置表示每60秒，如果数据发生了1000次以上的变动，Redis就会自动保存快照文件
save 60 1000
# 保存点可以设置多个，Redis的配置文件就默认设置了3个保存点
# 格式为：save <seconds> <changes>
# 可以设置多个。
save 900 1 #900秒后至少1个key有变动
save 300 10 #300秒后至少10个key有变动
save 60 10000 #60秒后至少10000个key有变动
```
## RDB的特点

优点

1. RDB 是一个二进制文件，以快照的形式保存 Redis 数据库的数据。它是一个紧凑的文件，它保存了 Redis 数据库的数据和元数据, 比较适合做数据备份。
2. 在数据量大的情况下，RDB恢复启动数据更快

缺点

1. 容易造成数据丢失，数据完整性保障不如 AOF，原因是RDB 是每隔一段时间进行保存，在保存间隔中丢失的数据没有保存下。
2. RDB 需要 for 子进程，数据较大的情况下对 Redis 服务器性能影响较大。 
   -  linux 系统中 fork 会拷贝进程的page table，数据量很大时耗时高, 并且当 redis 写入数据较大时，会造成**COW disaster**, 最糟糕情况下内存可能打到原先 2 倍。
   - rdb 文件产生的 page cache 也会驻留在系统中，当系统可用内存不足时，linux 会回收这部分 page cache，导致 redis 进程阻塞等待内存分配，引发服务抖动，对于这个问题 redis也添加了主动回收 page cache 的机制 https://github.com/redis/redis/pull/11248

# AOF
和 RDB 的快照保存方式不同，AOF 通过记录 redis 执行的命令来持久化。
AOF 持久化可以划分为三个过程：命令追加、文件写入和文件同步。

aof 文件长得很简单，下面的 aof 中记录了 select 2，和 set a b 两个命令：

![image-20250828115308078](/Users/c/Library/Application Support/typora-user-images/image-20250828115308078.png)

## 命令追加
当 redis 开启 AOF，每次执行一个写命令，redis 对会将其以 AOF 的格式追加到 aof_buf 的缓冲区中, 下面的函数在每次执行写命令的时候都会调用, 通过catAppendOnlyGenericCommand 将请求按照AOF 协议格式化后追加到 aof_buf 缓冲区中。
```c
/* Write the given command to the aof file.
 * dictid - dictionary id the command should be applied to,
 *          this is used in order to decide if a `select` command
 *          should also be written to the aof. Value of -1 means
 *          to avoid writing `select` command in any case.
 * argv   - The command to write to the aof.
 * argc   - Number of values in argv
 */
void feedAppendOnlyFile(int dictid, robj **argv, int argc) {
    sds buf = sdsempty();

    serverAssert(dictid == -1 || (dictid >= 0 && dictid < server.dbnum));

    /* Feed timestamp if needed */
    if (server.aof_timestamp_enabled) {
        sds ts = genAofTimestampAnnotationIfNeeded(0);
        if (ts != NULL) {
            buf = sdscatsds(buf, ts);
            sdsfree(ts);
        }
    }

    /* The DB this command was targeting is not the same as the last command
     * we appended. To issue a SELECT command is needed. */
    if (dictid != -1 && dictid != server.aof_selected_db) {
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }

    /* All commands should be propagated the same way in AOF as in replication.
     * No need for AOF-specific translation. */
    buf = catAppendOnlyGenericCommand(buf,argc,argv);

    /* Append to the AOF buffer. This will be flushed on disk just before
     * of re-entering the event loop, so before the client will get a
     * positive reply about the operation performed. */
    if (server.aof_state == AOF_ON ||
        (server.aof_state == AOF_WAIT_REWRITE && server.child_type == CHILD_TYPE_AOF))
    {
        server.aof_buf = sdscatlen(server.aof_buf, buf, sdslen(buf));
    }

    sdsfree(buf);
}
```
## 文件写入
在 server事件循环的`serverCron`中调用flushAppendOnlyFile 函数，定期将调用系统调用 write 将aof_buf 写入到文件中。

```c
// flushAppendOnlyFile
latencyStartMonitor(latency);
nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
latencyEndMonitor(latency);
```

## 文件同步

仅仅调用 `write` 写入文件并不能保证数据真正落盘，因为数据可能仍停留在内核页缓存中。Redis 需要通过 `fsync` 或 `fdatasync` 来保证持久化。

```c
try_fsync:
    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
    if (server.aof_no_fsync_on_rewrite && hasActiveChildProcess())
        return;

    /* Perform the fsync if needed. */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* redis_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        /* Let's try to get this data on the disk. To guarantee data safe when
         * the AOF fsync policy is 'always', we should exit if failed to fsync
         * AOF (see comment next to the exit(1) after write error above). */
        if (redis_fsync(server.aof_fd) == -1) {
            serverLog(LL_WARNING,"Can't persist AOF for fsync error when the "
              "AOF fsync policy is 'always': %s. Exiting...", strerror(errno));
            exit(1);
        }
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_last_incr_fsync_offset = server.aof_last_incr_size;
        server.aof_last_fsync = server.mstime;
        atomicSet(server.fsynced_reploff_pending, server.master_repl_offset);
    } else if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
               server.mstime - server.aof_last_fsync >= 1000) {
        if (!sync_in_progress) {
            aof_background_fsync(server.aof_fd);
            server.aof_last_incr_fsync_offset = server.aof_last_incr_size;
        }
        server.aof_last_fsync = server.mstime;
    }
```

fsync 可以通过 redis 设置中的`appendfsync`设置

| appendfsync 值 | flushAppendOnlyFile 行为                                     |
| -------------- | ------------------------------------------------------------ |
| always         | 每个事件循环都将 aof_buf 缓冲区中的内容写入 AOF 文件，并且调用 fsync() 将其同步到磁盘。这可以保证最好的数据持久性，但却会给系统带来极大的开销，其效率是三者中最慢的，但同时安全性也是最高的，即使宕机也只丢失一个事件循环中的数据。 |
| no             | 每个事件循环都将 aof_buf 缓冲区中的内容写入 AOF 文件，但不对其进行同步，何时同步至磁盘会让操作系统决定。这种模式下 AOF 的写入速度最快，不过因其会在系统缓存中积累一段时间的数据，所以同步时间为三者最长。一旦宕机将会丢失自上一次同步 AOF 文件起所有的数据。 |
| everysec       | 每个事件循环都将 aof_buf 缓冲区中的内容写入 AOF 文件，**Redis 还会每秒在子线程中执行一次 fsync()**。在实践中，推荐使用这种设置，一定程度上可以保证数据持久性，又不会明显降低 Redis 性能。 |

## AOF 重写

AOF 通过追加写入命令的方式持久化，同一个 key 可能有多次写入历史，而 AOF 需要将每次的写入都记录下来，因此相比于 RDB 其对磁盘空间消耗更大，AOF 文件会随着运行时间不断变大。为了解决当前问题，redis 引入了重写机制，即每过一段时间就对AOF 进行重写，去掉冗余的数据。

### 触发条件

用户可以通过`BGREWRITEAOF` 命令主动触发，redis 自身记录了当前 AOF 文件大小和上次 AOF 文件的大小，也可以设置自动触发条件：

```conf
auto-aof-rewrite-percentage 100	# 当前 AOF 文件大小相比上次重写后的大小增长了 100%（即翻倍）时
auto-aof-rewrite-min-size 64mb	# 当AOF 大小为 64MB时触发重写
```

### AOF如何重写

AOF 重写的主要函数是`rewriteAppendOnlyFileBackground` ，fork出子进程进行AOF重写
``````c
/* ----------------------------------------------------------------------------
 * AOF background rewrite
 * ------------------------------------------------------------------------- */

/* This is how rewriting of the append only file in background works:
 *
 * 1) The user calls BGREWRITEAOF
 * 2) Redis calls this function, that forks():
 *    2a) the child rewrite the append only file in a temp file.
 *    2b) the parent open a new INCR AOF file to continue writing.
 * 3) When the child finished '2a' exists.
 * 4) The parent will trap the exit code, if it's OK, it will:
 *    4a) get a new BASE file name and mark the previous (if we have) as the HISTORY type
 *    4b) rename(2) the temp file in new BASE file name
 *    4c) mark the rewritten INCR AOFs as history type
 *    4d) persist AOF manifest file
 *    4e) Delete the history files use bio
 */
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;

    if (hasActiveChildProcess()) return C_ERR;

    if (dirCreateIfMissing(server.aof_dirname) == -1) {
        serverLog(LL_WARNING, "Can't open or create append-only dir %s: %s",
            server.aof_dirname, strerror(errno));
        server.aof_lastbgrewrite_status = C_ERR;
        return C_ERR;
    }

    /* We set aof_selected_db to -1 in order to force the next call to the
     * feedAppendOnlyFile() to issue a SELECT command. */
    server.aof_selected_db = -1;
    flushAppendOnlyFile(1);
    if (openNewIncrAofForAppend() != C_OK) {
        server.aof_lastbgrewrite_status = C_ERR;
        return C_ERR;
    }

    if (server.aof_state == AOF_WAIT_REWRITE) {
        /* Wait for all bio jobs related to AOF to drain. This prevents a race
         * between updates to `fsynced_reploff_pending` of the worker thread, belonging
         * to the previous AOF, and the new one. This concern is specific for a full
         * sync scenario where we don't wanna risk the ACKed replication offset
         * jumping backwards or forward when switching to a different master. */
        bioDrainWorker(BIO_AOF_FSYNC);

        /* Set the initial repl_offset, which will be applied to fsynced_reploff
         * when AOFRW finishes (after possibly being updated by a bio thread) */
        atomicSet(server.fsynced_reploff_pending, server.master_repl_offset);
        server.fsynced_reploff = 0;
    }

    server.stat_aof_rewrites++;

    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {
        char tmpfile[256];

        /* Child */
        redisSetProcTitle("redis-aof-rewrite");
        redisSetCpuAffinity(server.aof_rewrite_cpulist);
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            serverLog(LL_NOTICE,
                "Successfully created the temporary AOF base file %s", tmpfile);
            sendChildCowInfo(CHILD_INFO_TYPE_AOF_COW_SIZE, "AOF rewrite");
            exitFromChild(0, 0);
        } else {
            exitFromChild(1, 0);
        }
    } else {
        /* Parent */
        if (childpid == -1) {
            server.aof_lastbgrewrite_status = C_ERR;
            serverLog(LL_WARNING,
                "Can't rewrite append only file in background: fork: %s",
                strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE,
            "Background append only file rewriting started by pid %ld",(long) childpid);
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);
        return C_OK;
    }
    return C_OK; /* unreached */
}
``````
他的主要逻辑

1. 打开一个新的 **INCR AOF 文件**（增量文件），继续处理客户端的写操作，将新的写命令记录到 AOF INCR文件中

2. 等待所有 **AOF fsync 线程**里的任务完成。

3. fork出子进程 

4. 【rewrite子进程】将当前数据库的完整状态写入**临时文件**：遍历所有键值对，生成对应的 Redis 命令

5. 【redis 主进程】继续处理客户端写操作，将新命令写到INCR AOF文件中

6. 【rewrite 子进程】完成rewrite，退出

7. 【redis主进程】在serverCron中通过`checkChildrenDone`的`waitpid` 检查子进程是否完成

8. 【redis 主线程】检查到子进程已完成， 

   - 将子进程生成的rewrite文件重命名为正式的AOF BASE文件。

   - 将临时的INCR AOF文件重命名为正式的INCR AOF文件。

   - 更新`aof_manifest`，持久化清单文件

   - 更新`aof_current_size`和`aof_rewrite_base_size`

   - 更新`fsynced_reploff`复制偏移量

   - 删除历史AOF文件

     下图是 bgrewriteaof 后的文件对比，可以看到 BASE AOF 和 INCR AOF 的编号已经更新

     ![image-20250829174541883](/Users/c/Library/Application Support/typora-user-images/image-20250829174541883.png)

# redis在持久化的最新动作



