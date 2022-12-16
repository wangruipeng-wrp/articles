---
title: Redis 持久化机制：RDB 和 AOF
abbrlink: 59553
date: 2021-09-13 00:21:49
copyright: false
description: RDB和AOF分别是Redis所提供的两种持久化方式，其中RDB是全量备份，也就是一个快照的方式，对应的AOF就是一个增量备份的方式，是一个日志备份的方式。
categories:
- Redis
---

RDB和AOF分别是Redis所提供的两种持久化方式，其中RDB是全量备份，也就是一个快照的方式，对应的AOF就是一个增量备份的方式，是一个日志备份的方式。

> 本文非原创，出自：[Redis进阶 - 持久化：RDB和AOF机制详解](https://pdai.tech/md/db/nosql-redis/db-redis-x-rdb-aof.html)

---

# RDB

触发 Redis 进行 RDB 备份的方式有两种，手动备份和自动备份。自动备份需要在配置文件种设置一些配置项，手动备份只需要手动向 Redis 控制台输入 `save` 命令或者是 `bgsave` 命令。

- `save`：阻塞当前Redis服务器，直到RDB过程完成为止，对于内存比较大的实例会造成长时间**阻塞**，线上环境不建议使用
- `bgsave`：Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短

**bgsave流程如下所示：**

![20220221142546](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20220221142546.png)

## 会自动触发RDB备份的四种情况

- redis.conf 中配置 `save m n`，即在 `m` 秒内有 `n` 次修改时，自动触发 bgsave 生成 rdb 文件
- 主从复制时，从节点要从主节点进行全量复制时也会触发 bgsave 操作，生成当时的快照发送到从节点
- reload 命令重新加载 redis 时也会触发 bgsave 操作
- 默认情况下执行 shutdown 命令时，如果没有开启 aof 持久化，那么也会触发 bgsave 操作

## redis.conf 中配置RDB备份

```
save 900 1  # 如果900秒内有1条Key信息发生变化，则进行快照
save ""     # 关闭RDB快照功能

dir /home/work/app/redis/data/  # rdb 文件保存路径
dbfilename dump.rdb             # rdb 文件名称

# 如果持久化出错，主进程是否停止写入
# 这个特性的主要目的是使运维人员在第一时间就发现Redis的运行错误，并进行解决
stop-writes-on-bgsave-error yes

# 是否压缩，将在字符串类型的数据被快照到磁盘文件时，启用 LZF 压缩算法
# Redis 官方的建议是请保持该选项设置为 yes
rdbcompression yes

# 导入时是否检查
rdbchecksum yes
```

rdbchecksum：从RDB快照功能的version 5 版本开始，一个64位的CRC冗余校验编码会被放置在RDB文件的末尾，以便对整个RDB文件的完整性进行验证。这个功能大概会多损失10%左右的性能，但获得了更高的数据可靠性。所以如果您的Redis服务需要追求极致的性能，就可以将这个选项设置为no。

## 拍摄快照的过程中如何保证数据一致性？

> 由于生产环境中我们为Redis开辟的内存区域都比较大（例如6GB），那么将内存中的数据同步到硬盘的过程可能就会持续比较长的时间，而实际情况是这段时间Redis服务一般都会收到数据写操作请求。那么如何保证数据一致性呢？

RDB 中的核心思路是 Copy-on-Write，来保证在进行快照操作的这段时间，需要压缩写入磁盘上的数据在内存中不会发生变化。在正常的快照操作中，一方面 Redis 主进程会 fork 一个新的快照进程专门来做这个事情，这样保证了 Redis 服务不会停止对客户端包括写请求在内的任何响应。另一方面这段时间发生的数据变化会以副本的方式存放在另一个新的内存区域，待快照操作结束后才会同步到原来的内存区域。 

举个例子：如果主线程对这些数据也都是读操作（例如图中的键值对 A），那么，主线程和 bgsave 子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对 C），那么，这块数据就会被复制一份，生成该数据的副本。然后，bgsave 子进程会把这个副本数据写入 RDB 文件，而在这个过程中，主线程仍然可以直接修改原来的数据。

![20220221144234](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20220221144234.png)

## RDB优缺点

**优点**

- RDB 文件是某个时间节点的快照，默认使用 LZF 算法进行压缩，压缩后的文件体积远远小于内存大小，适用于备份、全量复制等场景；
- Redis 加载 RDB 文件恢复数据要远远快于 AOF 方式；

**缺点**

- RDB 方式实时性不够，无法做到秒级的持久化；
- 每次调用 bgsave 都需要 fork 子进程，fork 子进程属于重量级操作，频繁执行成本较高；
- 版本兼容 RDB 文件问题；

---

# AOF

Redis 是 “写后” 日志，Redis 先执行命令，把数据写入内存，然后才记录日志。日志里记录的是 Redis 收到的每一条命令，这些命令是以文本形式保存。即先写内存，后写日志。

## 为什么采用写后日志？

Redis要求高性能，采用写后日志有两方面好处：

- 避免额外的检查开销：Redis 在向 AOF 里面记录日志的时候，并不会先去对这些命令进行语法检查。所以，如果先记日志再执行命令的话，日志中就有可能记录了错误的命令，Redis 在使用日志恢复数据时，就可能会出错。 
- 不会阻塞当前的写操作。

但这种方式存在潜在风险：

- 如果命令执行完成，写日志之前宕机了，会丢失数据。
- 主线程写磁盘压力大，导致写盘慢，阻塞后续操作。

## 如何实现 AOF

AOF日志记录Redis的每个写命令，步骤分为：命令追加（append）、文件写入（write）和文件同步（sync）。

- **命令追加：**当AOF持久化功能打开了，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器的 aof_buf 缓冲区。
- **文件写入和同步：**关于何时将 aof_buf 缓冲区的内容写入AOF文件中，Redis提供了三种写回策略：

## 三种写回策略

- **always 同步写回：** 每个写命令执行完，立马同步地将日志写回磁盘
- **everysec 每秒写回：** 每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘
- **no 操作系统控制的写回：** 每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘

**三种写会策略的优缺点：**

| 配置项 | 写回时机 | 优点 | 缺点 |
| :-- | :-- | :-- | :-- |
| always | 同步写回 | 可靠性高，数据基本不丢失 | 每个写命令都要落盘，性能影响较大 |
| everysec | 每秒写回 | 性能适中 | 宕机时丢失一秒内数据 |
| no | 操作系统控制写回 | 性能好 | 宕机丢失数据较多 |

> 为了提高文件写入效率，在现代操作系统中，当用户调用 write 函数，将一些数据写入文件时，操作系统通常会将数据暂存到一个内存缓冲区里，当缓冲区的空间被填满或超过了指定时限后，才真正将缓冲区的数据写入到磁盘里。
> 这样的操作虽然提高了效率，但也为数据写入带来了安全问题：如果计算机停机，内存缓冲区中的数据会丢失。为此，系统提供了 fsync、fdatasync 同步函数，可以强制操作系统立刻将缓冲区中的数据写入到硬盘里，从而确保写入数据的安全性。

## AOF重写

> AOF会记录每个写命令到AOF文件，随着时间越来越长，AOF文件会变得越来越大。如果不加以控制，会对Redis服务器，甚至对操作系统造成影响，而且AOF文件越大，数据恢复也越慢。为了解决AOF文件体积膨胀的问题，Redis提供AOF文件重写机制来对AOF文件进行“瘦身”。Redis通过创建一个新的AOF文件来替换现有的AOF，新旧两个AOF文件保存的数据相同，但新AOF文件没有了冗余命令。

![20220221161556](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20220221161556.png)

{% note default %}
**AOF重写流程**
{% endnote %}

AOF 重写过程是由后台进程 bgrewriteaof 来完成的。主线程 fork 出后台的 bgrewriteaof 子进程，fork 会把主线程的内存拷贝一份给 bgrewriteaof 子进程，这里面就包含了数据库的最新数据。然后，bgrewriteaof 子进程就可以在不影响主线程的情况下，逐一把拷贝的数据写成操作，记入重写日志。所以 **AOF 在重写时，在 fork 进程时是会阻塞住主线程的**。

1. 主线程fork出子进程重写aof日志
2. 子进程重写日志完成后，主线程追加aof日志缓冲
3. 替换日志文件

## redis.conf中配置AOF

```
appendonly no # appendonly 参数开启 AOF 持久化

# AOF持久化的文件名，默认是appendonly.aof
appendfilename "appendonly.aof"

# AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的
dir ./

appendfsync everysec # 同步写回策略

# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 加载aof出错如何处理
aof-load-truncated yes

# AOF 重写时，缓冲区满 32MB 写一次 fsync()，防止写硬盘时I/O突增。
aof-rewrite-incremental-fsync yes

# 重写触发配置，手动 “BGREWRITEAOF” 命令不受这两个参数限制
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

- `appendfsync`：这个参数项是AOF功能最重要的设置项之一，主要用于设置“真正执行”操作命令向AOF文件中同步的策略。与上节对应，appendfsync参数项可以设置三个值，分别是：always、everysec、no，默认的值为 everysec。

- `auto-aof-rewrite-percentage`：表示如果当前AOF文件的大小超过了上次重写后AOF文件的百分之多少后，就再次开始重写AOF文件。例如该参数值的默认设置值为100，意思就是如果AOF文件的大小超过上次AOF文件重写后的1倍，就启动重写操作。 

- `auto-aof-rewrite-min-size`：表示启动AOF文件重写操作的AOF文件最小大小，如果AOF文件大小低于这个值，则不会触发重写操作。

---

# RDB和AOF混合方式

> Redis 4.0 中提出了一个混合使用 AOF 日志和内存快照的方法。简单来说，内存快照以一定的频率执行，在两次快照之间，使用 AOF 日志记录这期间的所有命令操作。

这样一来，快照不用很频繁地执行，这就避免了频繁 fork 对主线程的影响。而且，AOF 日志也只用记录两次快照间的操作，也就是说，不需要记录所有操作了，因此，就不会出现文件过大的情况了，也可以避免重写开销。

如下图所示，T1 和 T2 时刻的修改，用 AOF 日志记录，等到第二次做全量快照时，就可以清空 AOF 日志，因为此时的修改都已经记录到快照中了，恢复时就不再用日志了。

![20220221162108](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20220221162108.png)

这个方法既能享受到 RDB 文件快速恢复的好处，又能享受到 AOF 只记录操作命令的简单优势, 实际环境中用的很多。

# 从持久化中恢复数据

![redis加载持久化数据顺序图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/redis加载持久化数据顺序图.png)

为什么会优先加载AOF呢？因为AOF保存的数据更完整，通过上面的分析我们知道AOF基本上最多损失1s的数据。