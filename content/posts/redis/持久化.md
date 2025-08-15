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
redis 作为内存数据库，数据的存储是放在内存中的，内存中的数据一旦断电或者重启就会丢失，因此需要持久化。

# 持久化的方式
redis 持久化方式有两种，分别是 RDB 和 AOF。
## RDB
RDB是Redis DataBase 的缩写，Redis 默认的持久化方式。RDB 持久化是将数据以**快照**的形式保存到磁盘上。 可以主动通过 save 或者 bgsave 命令来触发 RDB 持久化。