---
title: redis 后台IO线程设计
categories:
  - redis
slug: redis-background-IO
output:
  blogdown::html_page:
    toc: true
---

# bio模块介绍

redis 源码中 bio.c 是一个重要的模块，它实现了后台 IO（background IO） 机制，多线程异步执行一些耗时的操作，避免阻塞主线程。

# 什么操作需要BIO

```c
static char* bio_worker_title[] = {
    "bio_close_file",
    "bio_aof",
    "bio_lazy_free",
};

```

## AOF 刷盘操作

BIO_WORKER_AOF_FSYNC ： aof 操作需要定期调用`fsync`和`fdatasync`，这类的系统调用比较慢。

## 关闭文件描述符

BIO_WORKER_CLOSE_FILE：一些客户端关闭链接后、停止 aof 操作的涉及 `close`文件描述符。

## lazy free 机制

BIO_WORKER_LAZY_FREE：释放内存，例如：

1. 删除key 的时候，主线程仅仅把 key 从字典中删除，将释放内存的工作交给BIO 线程
2.  执行清理 db 命令如 `flushdb async` 时将清理旧字典的工作交给 BIO 线程

## 完成任务 (COMP_RQ)

为了更好地实现后台任务的同步，Redis 引入了完成请求（`COMP_RQ`）任务类型。这些任务主要用来通知主线程，某个特定的后台任务已经执行完成。通过这些完成请求，主线程可以阻塞等待后台任务的完成，确保执行的顺序和一致性。

# 实现原理

每一种任务都对应着一个队列，用来存放待处理的任务，每次有新的任务，就会放到对应的队列中。

```c
static list *bio_jobs[BIO_WORKER_NUM];
```

每一种任务都会启动一个专门的后台线程，不同任务互不影响

```c
static pthread_t bio_threads[BIO_WORKER_NUM];
```

再看几个代表性的 API

## bioInit

初始化后台线程和队列

```c
/* Initialize the background system, spawning the thread. */
void bioInit(void) {
    pthread_attr_t attr;
    pthread_t thread;
    size_t stacksize;
    unsigned long j;

    /* Initialization of state vars and objects */
    for (j = 0; j < BIO_WORKER_NUM; j++) {
        pthread_mutex_init(&bio_mutex[j],NULL);
        pthread_cond_init(&bio_newjob_cond[j],NULL);
        bio_jobs[j] = listCreate();
    }

    /* init jobs comp responses */
    bio_comp_list = listCreate();
    pthread_mutex_init(&bio_mutex_comp, NULL);

    /* Create a pipe for background thread to be able to wake up the redis main thread.
     * Make the pipe non blocking. This is just a best effort aware mechanism
     * and we do not want to block not in the read nor in the write half.
     * Enable close-on-exec flag on pipes in case of the fork-exec system calls in
     * sentinels or redis servers. */
    if (anetPipe(job_comp_pipe, O_CLOEXEC|O_NONBLOCK, O_CLOEXEC|O_NONBLOCK) == -1) {
        serverLog(LL_WARNING,
                  "Can't create the pipe for bio thread: %s", strerror(errno));
        exit(1);
    }

    /* Register a readable event for the pipe used to awake the event loop on job completion */
    if (aeCreateFileEvent(server.el, job_comp_pipe[0], AE_READABLE,
                          bioPipeReadJobCompList, NULL) == AE_ERR) {
        serverPanic("Error registering the readable event for the bio pipe.");
    }

    /* Set the stack size as by default it may be small in some system */
    pthread_attr_init(&attr);
    pthread_attr_getstacksize(&attr,&stacksize);
    if (!stacksize) stacksize = 1; /* The world is full of Solaris Fixes */
    while (stacksize < REDIS_THREAD_STACK_SIZE) stacksize *= 2;
    pthread_attr_setstacksize(&attr, stacksize);

    /* Ready to spawn our threads. We use the single argument the thread
     * function accepts in order to pass the job ID the thread is
     * responsible for. */
    for (j = 0; j < BIO_WORKER_NUM; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING, "Fatal: Can't initialize Background Jobs. Error message: %s", strerror(errno));
            exit(1);
        }
        bio_threads[j] = thread;
    }
}

```

## bioCreateXXXJob

创建某一任务，目前共有bioCreateLazyFreeJob、bioCreateCompRq、bioCreateCloseJob、bioCreateCloseAofJob、bioCreateFsyncJob 这六种。

以下面的bioCreateCloseJob 为例，创建了一个 job 并将其提交。

```c
void bioCreateCloseJob(int fd, int need_fsync, int need_reclaim_cache) {
    bio_job *job = zmalloc(sizeof(*job));
    job->fd_args.fd = fd;
    job->fd_args.need_fsync = need_fsync;
    job->fd_args.need_reclaim_cache = need_reclaim_cache;

    bioSubmitJob(BIO_CLOSE_FILE, job);
}
```

## bioSubmitJob

加入到对应的队列中并唤醒线程。

```c
void bioSubmitJob(int type, bio_job *job) {
    job->header.type = type;
    unsigned long worker = bio_job_to_worker[type];
    pthread_mutex_lock(&bio_mutex[worker]);
    listAddNodeTail(bio_jobs[worker],job);
    bio_jobs_counter[type]++;
    pthread_cond_signal(&bio_newjob_cond[worker]);
    pthread_mutex_unlock(&bio_mutex[worker]);
}
```

## bioProcessBackgroundJobs

后台线程的核心循环任务，主要工作是从队列中取出并执行

```c
...
while(1) {
    listNode *ln;

    /* The loop always starts with the lock hold. */
    if (listLength(bio_jobs[worker]) == 0) {
        pthread_cond_wait(&bio_newjob_cond[worker], &bio_mutex[worker]);
        continue;
    }
    /* Get the job from the queue. */
    ln = listFirst(bio_jobs[worker]);
    job = ln->value;
    /* It is now possible to unlock the background system as we know have
     * a stand alone job structure to process.*/
    pthread_mutex_unlock(&bio_mutex[worker]);

    /* Process the job accordingly to its type. */
    int job_type = job->header.type;

    if (job_type == BIO_CLOSE_FILE) {
        if (job->fd_args.need_fsync &&
            redis_fsync(job->fd_args.fd) == -1 &&
            errno != EBADF && errno != EINVAL)
        {
            serverLog(LL_WARNING, "Fail to fsync the AOF file: %s",strerror(errno));
        }
        if (job->fd_args.need_reclaim_cache) {
            if (reclaimFilePageCache(job->fd_args.fd, 0, 0) == -1) {
                serverLog(LL_NOTICE,"Unable to reclaim page cache: %s", strerror(errno));
            }
        }
        close(job->fd_args.fd);
    } 
```
