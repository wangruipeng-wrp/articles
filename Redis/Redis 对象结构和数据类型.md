---
title: Redis 对象结构和数据类型
abbrlink: 35006
date: 2022-09-14 16:41:33
categories:
- Redis
---

> 本文转载自：[小林Coding-Redis 常见数据类型和应用场景](https://xiaolincoding.com/redis/data_struct/command.html)

Redis 是一个键值对数据库，它的键值对中的 key 就是字符串对象，value 可以是字符串对象也可以是其他的集合数据类型对象，比如：List、Hash、Set、Zset 等。

Redis 中常见的数据类型有五种：String（字符串）、Hash（哈希）、List（列表）、Set（集合）、Zset（有序集合）。

随着 Redis 版本更新，后面又支持了四种：BitMap（2.2 版新增）、HyperLogLog（2.8 版新增）、GEO（3.2 版新增）、Stream（5.0 版新增）。

# Redis 对象结构

Redis 是使用了一个**哈希表**保存所有键值对，哈希表的最大好处就是让我们可以用 O(1) 的时间复杂度来快速查找到键值对。哈希表其实就是一个数组，数组中的元素叫做哈希桶。

哈希桶存放的是指向键值对数据的指针，这样通过指针就能找到键值对数据。因为键值对的 value 可以是字符串对象或者集合对象，所以键值对中并不直接保存值本身，而是保存了 `void * key` 和 `void * value` 指针，分别指向了实际的键对象和值对象，这样一来，即使值是集合数据，也可以通过 value 指针找到。

**Redis对象结构示意图：**

![Redis对象结构示意图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Redis对象结构示意图.jpg)

以上三个属性是 Redis 对象中重要的三个属性，Redis 对象还有其他的一些属性，这里就先不展开。

- **type**：标识该对象是什么类型的对象（String、List、Hash、Set、Zset）
- **encoding**：标识该对象使用了哪种底层的数据结构
- **ptr**：指向底层数据结构的指针

# String

String 是最基本的 key-value 结构，key 是唯一标识，value 是具体的值，value其实不仅是字符串， 也可以是数字（整数或浮点数），value 最多可以容纳的数据长度是 **512M**。底层数据结构是 int 和 SDS（简单动态字符串）。

SDS 是 Redis 内部实现的字符串对象，并没有采用 C语言 的字符串表示。由 Redis 内部实现的 SDS 优势如下：

1. SDS 可以保存二进制数据，因为 SDS 采用 len 属性的值来表示字符串是否结束。
2. SDS 获取字符串长度的时间复杂度是 O(1)，因为 SDS 采用 len 属性记录字符串长度
3. SDS 拼接字符串不会造成缓冲区溢出，是安全的，因为 SDS 在拼接字符串之前会检查 SDS 空间是否满足要求，空间不够会自动扩容，所以不会导致缓冲区溢出的问题。

字符串对象的内部编码（encoding）有 3 种 ：**int、embstr 和 raw**。

- **int**：如果保存的是可以用 long 类型表示的整数值，那么直接将值保存在 ptr 属性里面即可，编码设置为 int。

- **embstr 和 raw**：保存的是字符串，底层的数据结构都是 SDS，如果是较短的字符串那么将编码设置为 embstr，如果是较长的字符串那么将编码设置为 raw。

embstr 和 raw 的边界在不同的 Redis 版本中是不相同的。embstr 会通过一次内存分配函数来分配一块连续的内存空间来保存 redisObject 和 SDS，而 raw 编码会通过调用两次内存分配函数来分别分配两块空间来保存redisObject 和 SDS。Redis 这样做会有很多好处：

- embstr 编码在创建和释放对象时只需要调用一次内存函数。
- embstr 所有的数据都保存在一块连续的内存，可以更好的利用 CPU 缓存提高性能。

但 embstr 也是有缺点的，如果需要重新分配字符串内存时，整个 redisObject 和 SDS 都需要重新分配空间，**所以 embstr 编码的字符串对象是只读的**，我们对 embstr 编码的字符串对象执行的任何修改，程序都会先将对象的编码从 embstr 转换成 raw，然后再执行修改。

**String 类型的使用场景：**缓存对象、常规计数、分布式锁、分布式服务之间共享 Session 信息

# List

List 列表是简单的字符串列表，按照插入顺序排序，可以从头部或尾部向 List 列表添加元素。列表的最大长度为 `2^32 - 1`，也即每个列表支持超过 40 亿个元素。在 Redis 3.2 版本之后，List 数据类型底层数据结构只由 quicklist 实现，替代了双向链表和压缩列表。

**List 类型的使用场景：**消息队列

消息队列在存取消息时，必须要满足三个需求，分别是：**消息保序、处理重复的消息和保证消息可靠性**。

{% note default %}
**如何满足消息保序需求？**
{% endnote %}

List 本身就是按先进先出的顺序对数据进行存取的，所以，如果使用 List 作为消息队列保存消息的话，就已经能满足消息保序的需求了。

List 可以使用 LPUSH + RPOP （或者反过来，RPUSH + LPOP）命令实现消息队列。不过这种方式在消费者读取数据时，有一个潜在的性能风险点：

生产者往 List 中写入消息时并不会通知消费者去消费消息，消费者只能不断的尝试从 List 中去取消息，这会导致消费者的 CPU 一直消耗在 RPOP 命令上，带来不必要的性能损失。

为了解决这个问题，Redis提供了 BRPOP 命令。**BRPOP命令也称为阻塞式读取**，客户端在没有读到队列数据时，自动阻塞，直到有新的数据写入队列，再开始读取新数据。和消费者程序自己不停地调用RPOP命令相比，这种方式能节省CPU开销。

{% note default %}
**如何处理重复的消息？**
{% endnote %}

需要为每一条消息生成一个 ID 号，消费者记录自己消费过的 ID，取出消息时先判断自己是否消费过此消息，如果消费过则不再处理。

{% note default %}
**如何保证消息可靠性？**
{% endnote %}

当消费者程序从 List 中读取一条消息后，List 就不会再留存这条消息了。所以，如果消费者程序在处理消息的过程出现了故障或宕机，就会导致消息没有处理完成，那么，消费者程序再次启动后，就没法再次从 List 中读取消息了。

为了留存消息，List 类型提供了 `BRPOPLPUSH` 命令，BRPOPLPUSH 命令从列表中取出最后一个元素，并插入到另外一个列表的头部；如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。

> BRPOPLPUSH 命令：`BRPOPLPUSH L1 L2 10`，从 L1 取出一条消息返回并存入 L2 中，阻塞时间为 10s。

使用 BRPOPLPUSH 命令后，如果消费者程序读了消息但没能正常处理，等它重启后，就可以从备份 List 中重新读取消息并进行处理了。

# Hash

Hash 是一个键值对（key - value）集合，特别适合用于存储对象。其中 value 的形式如：`value=[{field1, value1},...{fieldN, valueN}]`。在 Redis7.0 以后底层的数据结构改为 listpack，不再使用压缩列表或哈希表。

**应用场景：**缓存对象、购物车（以 用户ID 为 key，商品ID 为 field，商品数量为 value）

```bash
# 缓存 UID:1 对象
> HSET UID:1 name admin age 18
(integer) 2

# 获取 UID:1 对象的姓名、年龄等属性值
> HGET UID:1 name
"admin"
> HGET UID:1 age
"18"
```

```bash
# 创建购物车
> HSET SHOPPINGCART:UID:01 1001 1 1002 2 1003 10
(integer) 3

# 获取购物车信息
> HGETALL SHOPPINGCART:UID:01
1) "1001"
2) "1"
3) "1002"
4) "2"
5) "1003"
6) "10"
```

# Set

Set 类型是一个无序并唯一的集合，它的存储顺序不会按照插入的先后顺序进行存储。

一个集合最多可以存储 `2^32-1` 个元素。概念和数学中个的集合基本类似，可以交集，并集，差集等等，所以 Set 类型除了支持集合内的增删改查，同时还支持多个集合取交集、并集、差集。

但是 Set 的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，可能会导致 Redis 实例阻塞。

在主从集群中，为了避免主库因为 Set 做聚合计算（交集、差集、并集）时导致主库被阻塞，我们可以选择一个从库完成聚合统计，或者把数据返回给客户端，由客户端来完成聚合统计。

{% note primary %}
**Set 集合运算操作**
{% endnote %}

```bash
# 交集运算
SINTER key [key ...]
# 将交集结果存入新集合destination中
SINTERSTORE destination key [key ...]

# 并集运算
SUNION key [key ...]
# 将并集结果存入新集合destination中
SUNIONSTORE destination key [key ...]

# 差集运算
SDIFF key [key ...]
# 将差集结果存入新集合destination中
SDIFFSTORE destination key [key ...]
```

**应用场景：**点赞、共同关注、抽奖活动。

{% note default %}
**点赞**
{% endnote %}

Set 集合可以保证一个用户只能点一个赞，这里举例子一个场景，key 是文章ID，value 是用户ID。

```bash
# uid:1 用户对文章 article:1 点赞
> SADD article:1 uid:1
(integer) 1
# uid:2 用户对文章 article:1 点赞
> SADD article:1 uid:2
(integer) 1
# uid:3 用户对文章 article:1 点赞
> SADD article:1 uid:3
(integer) 1

# uid:1 取消了对 article:1 文章点赞
> SREM article:1 uid:1
(integer) 1

# 获取 article:1 文章所有点赞用户
> SMEMBERS article:1
1) "uid:3"
2) "uid:2"

# 获取 article:1 文章的点赞用户数量
> SCARD article:1
(integer) 2

# 判断用户 uid:1 是否对文章 article:1 点赞了
> SISMEMBER article:1 uid:1
(integer) 0  # 返回0说明没点赞，返回1则说明点赞了
```

{% note default %}
**共同关注**
{% endnote %}

Set 类型支持交集运算，所以可以用来计算共同关注的好友、公众号等。key 可以是用户id，value 则是已关注的公众号的id。

```bash
# uid:1 用户关注公众号 id 为 5、6、7、8、9
> SADD uid:1 5 6 7 8 9
(integer) 5
# uid:2  用户关注公众号 id 为 7、8、9、10、11
> SADD uid:2 7 8 9 10 11
(integer) 5

# uid:1 和 uid:2 共同关注的公众号
> SINTER uid:1 uid:2
1) "7"
2) "8"
3) "9"

# 给 uid:2 推荐 uid:1 关注的公众号
> SDIFF uid:1 uid:2
1) "5"
2) "6"

# 给 uid:1 推荐 uid:2 关注的公众号
> SDIFF uid:2 uid:1
1) "10"
2) "11"

# 验证某个公众号是否被 uid:1 或 uid:2 关注
> SISMEMBER uid:1 5
(integer) 1 # 返回1，说明关注了
> SISMEMBER uid:2 5
(integer) 0 # 返回0，说明没关注
```

{% note default %}
**抽奖活动**
{% endnote %}

存储某活动中中奖的用户名 ，Set 类型因为有去重功能，可以保证同一个用户不会中奖两次。

如果允许重复中奖，使用 SRANDMEMBER 命令。如果不允许重复中奖，使用 SPOP 命令。

```bash
# 从集合key中随机选出count个元素，元素不从key中删除
SRANDMEMBER key [count]

# 从集合key中随机选出count个元素，元素从key中删除
SPOP key [count]
```

# Zset

Zset 有序集合相比于 Set 集合多了一个排序属性 score（分值），对于有序集合 ZSet 来说，每个存储元素相当于有两个值组成的，一个是有序集合的元素值，一个是元素排序值。

有序集合保留了集合不能有重复成员的特性（分值可以重复），但不同的是，有序集合中的元素可以排序。

在 Redis 7.0 中，Zset 有序集合的底层数据结构就交由 listpack 数据结构来实现了，压缩列表数据结构就被废弃了。

{% note primary %}
**常用命令**
{% endnote %}

```bash
# 往有序集合key中加入带分值元素
ZADD key score member [[score member]...]   
# 往有序集合key中删除元素
ZREM key member [member...]                 
# 返回有序集合key中元素member的分值
ZSCORE key member
# 返回有序集合key中元素个数
ZCARD key 

# 为有序集合key中元素member的分值加上increment
ZINCRBY key increment member

# 正序获取有序集合key从start下标到stop下标的元素
ZRANGE key start stop [WITHSCORES]
# 倒序获取有序集合key从start下标到stop下标的元素
ZREVRANGE key start stop [WITHSCORES]

# 返回有序集合中指定分数区间内的成员，分数由低到高排序。
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]

# 返回指定成员区间内的成员，按字典正序排列, 分数必须相同。
ZRANGEBYLEX key min max [LIMIT offset count]
# 返回指定成员区间内的成员，按字典倒序排列, 分数必须相同
ZREVRANGEBYLEX key max min [LIMIT offset count]

# 并集计算(相同元素分值相加)，numberkeys一共多少个key，结果保存至destkey集合
ZUNIONSTORE destkey numberkeys key [key...] 
# 交集计算(相同元素分值相加)，numberkeys一共多少个key，结果保存至destkey集合
ZINTERSTORE destkey numberkeys key [key...]
```

**应用场景：**排行榜、电话或者姓名等排序

{% note default %}
**排行榜**
{% endnote %}

以文章点赞数排行榜为例：

```bash
# arcticle:1 文章获得了200个赞
> ZADD user:1:ranking 200 arcticle:1
(integer) 1
# arcticle:2 文章获得了40个赞
> ZADD user:1:ranking 40 arcticle:2
(integer) 1
# arcticle:3 文章获得了100个赞
> ZADD user:1:ranking 100 arcticle:3
(integer) 1
# arcticle:4 文章获得了50个赞
> ZADD user:1:ranking 50 arcticle:4
(integer) 1
# arcticle:5 文章获得了150个赞
> ZADD user:1:ranking 150 arcticle:5
(integer) 1
```

{% note default %}
**电话排序**
{% endnote %}

使用有序集合的 ZRANGEBYLEX 或 ZREVRANGEBYLEX 可以帮助我们实现电话号码或姓名的排序，我们以 ZRANGEBYLEX （返回指定成员区间内的成员，按 key 正序排列，分数必须相同）为例。

注意：不要在分数不一致的 SortSet 集合中去使用 ZRANGEBYLEX和 ZREVRANGEBYLEX 指令，因为获取的结果会不准确。

```bash
> ZADD phone 0 13100111100 0 13110114300 0 13132110901 
(integer) 3
> ZADD phone 0 13200111100 0 13210414300 0 13252110901 
(integer) 3
> ZADD phone 0 13300111100 0 13310414300 0 13352110901 
(integer) 3

# 获取所有电话号码
> ZRANGEBYLEX phone - +
1) "13100111100"
2) "13110114300"
3) "13132110901"
4) "13200111100"
5) "13210414300"
6) "13252110901"
7) "13300111100"
8) "13310414300"
9) "13352110901"

# 获取 132 号段的号码
> ZRANGEBYLEX phone [132 (133
1) "13200111100"
2) "13210414300"
3) "13252110901"
```

# BitMap

Bitmap，即位图，是一串连续的二进制数组（0和1），可以通过偏移量（offset）定位元素。BitMap 通过最小的单位bit来进行 `0|1` 的设置，表示某个元素的值或者状态，时间复杂度为 O(1)。使用它进行储存将非常节省空间，特别适合一些数据量大且使用二值统计的场景。

![BitMap存储示意图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/BitMap存储示意图.png)

Bitmap 本身是用 String 类型作为底层数据结构实现的一种统计二值状态的数据类型。

String 类型是会保存为二进制的字节数组，所以，Redis 就把字节数组的每个 bit 位利用起来，用来表示一个元素的二值状态，你可以把 Bitmap 看作是一个 bit 数组。

{% note primary %}
**常用命令**
{% endnote %}

```bash
# 设置值，其中value只能是 0 和 1
SETBIT key offset value

# 获取值
GETBIT key offset

# 获取指定范围内值为 1 的个数
# start 和 end 以字节为单位
BITCOUNT key start end
```

```java
# BitMap间的运算
# operations 位移操作符，枚举值
#   AND 与运算 &
#   OR 或运算 |
#   XOR 异或 ^
#   NOT 取反 ~
# result 计算的结果，会存储在该key中
# key1 … keyn 参与运算的key，可以有多个，空格分割，NOT 运算只能一个key
# 当 BITOP 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 0。
# 返回值是保存到 destkey 的字符串的长度（以字节byte为单位），和输入 key 中最长的字符串长度相等。
BITOP [operations] [result] [key1] [keyn…]

# 返回指定key中第一次出现指定value(0/1)的位置
BITPOS [key] [value]
```

**应用场景：**签到统计、判断用户登录状态、连续签到用户统计

{% note default %}
**连续签到用户统计**
{% endnote %}

以统计七天连续签到打卡的用户为例：

把每天的日期作为 Bitmap 的 key，userId 作为 offset，若是打卡则将 offset 位置的 bit 设置成 1。key 对应的集合的每个 bit 位的数据则是一个用户在该日期的打卡记录。

一共有 7 个这样的 Bitmap，如果我们能对这 7 个 Bitmap 的对应的 bit 位做 『与』运算。同样的 UserID offset 都是一样的，当一个 userID 在 7 个 Bitmap 对应对应的 offset 位置的 bit = 1 就说明该用户 7 天连续打卡。

结果保存到一个新 Bitmap 中，我们再通过 BITCOUNT 统计 bit = 1 的个数便得到了连续打卡 7 天的用户总数了。

# HyperLogLog

HyperLogLog 是一个用来做基数估计的数据类型，什么是基数估计呢？

比如现有数据集：{1, 3, 5, 7, 5, 7, 8}，其中构成这个集合的基本元素是：{1, 3, 5, 7, 8}，总共有 5 个，那么基数就是 5。基数估计就是在可接受的误差范围内，快速计算基数。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 `2^64` 个不同元素的基数，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 就非常节省空间。

不过，有一点需要你注意一下，HyperLogLog 的统计规则是基于概率完成的，所以它给出的统计结果是有一定误差的，标准误算率是 0.81%。

{% note primary %}
**HyperLogLog 使用命令：**
{% endnote %}

```bash
# 添加指定元素到 HyperLogLog 中
PFADD key element [element ...]

# 返回给定 HyperLogLog 的基数估算值。
PFCOUNT key [key ...]

# 将多个 HyperLogLog 合并为一个 HyperLogLog
PFMERGE destkey sourcekey [sourcekey ...]
```

**应用场景：**百万级网页 UV 计数

在统计 UV 时，你可以用 PFADD 命令（用于向 HyperLogLog 中添加新元素）把访问页面的每个用户都添加到 HyperLogLog 中。

接下来，就可以用 PFCOUNT 命令直接获得 page1 的 UV 值了，这个命令的作用就是返回 HyperLogLog 的统计结果。

# Stream

Stream 是 Redis 5.0 版本新增加的数据类型，是 Redis 专门为消息队列设计的数据类型。在 Redis 5.0 Stream 没出来之前，消息队列的实现方式都有着各自的缺陷，例如：

- 发布订阅模式，不能持久化也就无法可靠的保存消息，并且对于离线重连的客户端不能读取历史消息的缺陷；
- List 实现消息队列的方式不能重复消费，一个消息消费完就会被删除，而且生产者需要自行实现全局唯一 ID。

基于以上问题，Redis 5.0 便推出了 Stream 用于完美地实现消息队列，它支持消息的持久化、支持自动生成全局唯一 ID、支持 ack 确认消息的模式、支持消费组模式等，让消息队列更加的稳定和可靠。

{% note default %}
**基本操作：生产和消费**
{% endnote %}

```bash
# * 表示让 Redis 为插入的数据自动生成一个全局唯一的 ID
# 往名称为 mymq 的消息队列中插入一条消息，消息的键是 name，值是 zhangsan
> XADD mymq * name zhangsan
"1663241868161-0"
```

插入成功后会返回全局唯一的 ID："1663241868161-0"。消息的全局唯一 ID 由两部分组成：

- 第一部分“1663241868161”是数据插入时，以毫秒为单位计算的当前服务器时间；
- 第二部分表示插入消息在当前毫秒内的消息序号，这是从 0 开始编号的。例如，“1663241868161-0”就表示在“1663241868161”毫秒内的第 1 条消息。

消费者通过 XREAD 命令从消息队列中读取消息时，可以指定一个消息 ID，并从这个消息 ID 的下一条消息开始进行读取（注意是输入消息 ID 的下一条信息开始读取，不读取输入ID的消息）

```bash
# 这里输入的毫秒时间戳是刚才生成的ID的前一毫秒
> XREAD STREAMS mymq 1663241868160-0
1) 1) "mymq"
   2) 1) 1) "1663241868161-0"
         2) 1) "name"
            2) "zhangsan"
```

如果想要实现阻塞读（当没有数据时，阻塞住），可以调用 XRAED 时设定 BLOCK 配置项，实现类似于 BRPOP 的阻塞读取操作。

```bash
# 10000 = 10s，命令最后的“$”符号表示读取最新的消息
> XREAD BLOCK 10000 STREAMS mymq $
(nil)
(10.00s)
```

{% note default %}
**消费组**
{% endnote %}

Stream 可以以使用 XGROUP 创建消费组，创建消费组之后，Stream 可以使用 XREADGROUP 命令让消费组内的消费者读取消息。

创建两个消费组，这两个消费组消费的消息队列是 mymq，都指定从第一条消息开始读取：

```bash
# 创建一个名为 group1 的消费组，0-0 表示从第一条消息开始读取。
> XGROUP CREATE mymq group1 0-0
OK
# 创建一个名为 group2 的消费组，0-0 表示从第一条消息开始读取。
> XGROUP CREATE mymq group2 0-0
OK
```

消费组 group1 内的消费者 consumer1 从 mymq 消息队列中读取所有消息的命令如下：

```bash
# 命令最后的参数“>”，表示从第一条尚未被消费的消息开始读取。
> XREADGROUP GROUP group1 consumer1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1654254953808-0"
         2) 1) "name"
            2) "xiaolin"
```

消息队列中的消息一旦被消费组里的一个消费者读取了，就不能再被该消费组内的其他消费者读取了，即同一个消费组里的消费者不能消费同一条消息。

不同消费组的消费者可以消费同一条消息（但是有前提条件，创建消息组的时候，不同消费组指定了相同位置开始读取消息）

使用消费组的目的是让组内的多个消费者共同分担读取消息，所以，我们通常会让每个消费者读取部分消息，从而实现消息读取负载在多个消费者间是均衡分布的。

```bash
# 让 group2 中的 consumer1 从 mymq 消息队列中消费一条消息
> XREADGROUP GROUP group2 consumer1 COUNT 1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1663243059501-0"
         2) 1) "name"
            2) "1"
# 让 group2 中的 consumer2 从 mymq 消息队列中消费一条消息
> XREADGROUP GROUP group2 consumer2 COUNT 1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1663243061599-0"
         2) 1) "name"
            2) "2"
# 让 group2 中的 consumer3 从 mymq 消息队列中消费一条消息
> XREADGROUP GROUP group2 consumer3 COUNT 1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1663243062597-0"
         2) 1) "name"
            2) "3"
```

{% note default %}
**消息确认**
{% endnote %}

Streams 会自动使用内部队列（也称为 PENDING List）留存消费组里每个消费者读取的消息，直到消费者使用 XACK 命令通知 Streams “消息已经处理完成”。

消费确认增加了消息的可靠性，一般在业务处理完成之后，需要执行 XACK 命令确认消息已经被消费完成，整个流程的执行如下图所示：

![Stream消息确认流程图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Stream消息确认流程图.png)

如果消费者没有成功处理消息，它就不会给 Streams 发送 XACK 命令，消息仍然会留存。此时，消费者可以在重启后，用 XPENDING 命令查看已读取、但尚未确认处理完成的消息。

```bash
> XPENDING mymq group1
1) (integer) 3
2) "1663245004996-0" # 表示 group1 中所有消费者读取的消息最小 ID
3) "1663245010578-0" # 表示 group1 中所有消费者读取的消息最大 ID
4) 1) 1) "consumer1"
      2) "1"
   2) 1) "consumer2"
      2) "1"
   3) 1) "consumer3"
      2) "1"
```

如果想查看某个消费者具体读取了哪些数据，可以执行下面的命令：

```bash
# 查看 group1 里 consumer2 从 mymq 消息队列中读取未确认哪些消息
> XPENDING mymq group1 - + 10 consumer2
1) 1) "1663245009507-0"
   2) "consumer2"
   3) (integer) 276418
   4) (integer) 1
```

group1 里 consumer2 应答 1663245009507-0：

```bash
# 应答
> XACK mymq GROUP1 1663245009507-0
(integer) 1

# 应答成功后没有未确认消息了
> XPENDING mymq GROUP1 - + 10 consumer2
(empty array)
```

{% note default %}
**对于 MQ 来说，为什么 Redis 显得不专业？**
{% endnote %}

1. Redis 在以下 2 个场景下，可能会导致数据丢失：
    - AOF 持久化配置为每秒写盘，但这个写盘过程是异步的，Redis 宕机时会存在数据丢失的可能
    - 主从复制也是异步的，主从切换时，也存在丢失数据的可能 (opens new window)。

2. Redis Stream 消息不可堆积
    - Redis 的数据都存储在内存中，这就意味着一旦发生消息积压，则会导致 Redis 的内存持续增长，如果超过机器内存上限，就会面临被 OOM 的风险。