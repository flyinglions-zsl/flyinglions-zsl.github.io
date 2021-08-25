---
title: redis学习
date: 2021-07-15 22:46:33
tags: 
	- redis
categories:
	- redis

---

基于Redis6.0版本

# 基本相关指令

| 指令              | 说明              |
| ----------------- | ----------------- |
| keys *            | 查看所有key       |
| flushdb           | 删除所有key       |
| sort key asc/desc | 对key进行排序查询 |

# 数据结构分类

## 字符串string

### 常用操作指令

| 指令                             | 说明                                           |
| -------------------------------- | ---------------------------------------------- |
| SET  key  value                  | 存入字符串键值对                               |
| MSET  key  value [key value ...] | 批量存储字符串键值对                           |
| SETNX  key  value                | 存入一个不存在的字符串键值对                   |
| GET  key                         | 获取一个字符串键值                             |
| MGET  key  [key ...]             | 批量获取字符串键值                             |
| DEL  key  [key ...]              | 删除一个键                                     |
| EXPIRE  key  seconds             | 设置一个键的过期时间(秒)                       |
| INCR  key                        | 将key中储存的数字值加1(原子操作)               |
| DECR  key                        | 将key中储存的数字值减1(原子操作)               |
| INCRBY  key  increment           | 将key所储存的数字加上指定的increment(原子操作) |
| DECRBY  key  decrement           | 将key所储存的数字减去指定的decrement(原子操作) |

```
127.0.0.1:6379> set name:1 true ex/px 10 nx/xx
OK
在 Redis 中设置值，默认，不存在则创建，存在则修改 
参数： 
ex：过期时间（秒） 
px：过期时间（毫秒）
nx：只在键不存在时，才对键进行设置操作。 SET key value NX 效果等同于 SETNX key value 
xx：只在键已经存在时，才对键进行设置操作。
```



### 应用场景

#### 单个值的缓存

```
127.0.0.1:6379> set name:6 666
OK
127.0.0.1:6379> get name:6
"666"
```

#### 对象的缓存

```
127.0.0.1:6379> set people:xx "{\"name\":\"xx\",\"age\",\"1\"}"
OK
127.0.0.1:6379> get people:xx
"{\"name\":\"xx\",\"age\",\"1\"}"

127.0.0.1:6379>  mset people1:age 1 people2:age 2 people3:age 3
OK
127.0.0.1:6379> mget people1:age people2:age people3:age
1) "1"
2) "2"
3) "3"
```

#### 计数器

```
127.0.0.1:6379> incr ld
(integer) 1
127.0.0.1:6379> incr ld
(integer) 2
127.0.0.1:6379> incr ld
(integer) 3
127.0.0.1:6379> incr ld
(integer) 4
127.0.0.1:6379> get ld
"4"
127.0.0.1:6379> decr ld
(integer) 3
127.0.0.1:6379> 
127.0.0.1:6379> decr ld
(integer) 2
127.0.0.1:6379> get ld
"2"
127.0.0.1:6379> incrby ld 2
(integer) 4
127.0.0.1:6379> incrby ld 2
(integer) 6
127.0.0.1:6379> get ld
"6"
```

#### 分布式锁

可参考：[redis分布式锁思路实现](https://ifeve.com/基于redis的分布式锁/)

还涉及到锁过期，刷新过期时间保证任务运行时间足够等

```
127.0.0.1:6379> setnx mykey true
(integer) 1
127.0.0.1:6379> setnx mykey true//再次set，返回0，可代表获取锁失败
(integer) 0
127.0.0.1:6379> del mykey
(integer) 1
127.0.0.1:6379> setnx mykey true
(integer) 1
127.0.0.1:6379> set mykey true ex 30 nx//当key不存在时，设置30s过期
OK
```



## 哈希hash

### 常用操作指令

| 指令                                       | 说明                                      |
| ------------------------------------------ | ----------------------------------------- |
| HSET  key  field value                     | 存入一个哈希表key的键值                   |
| HSETNX  key  field  value                  | 存储一个不存在的哈希表key的键值           |
| HMSET  key  field  value [field value ...] | 在一个哈希表key中存储多个键值对           |
| HGET  key  field                           | 获取哈希表key对应的field键值              |
| HMGET  key  field  [field ...]             | 批量获取哈希表key中多个field键值          |
| HDEL  key  field  [field ...]              | 删除哈希表key中的field键值                |
| HLEN  key                                  | 返回哈希表key中field的数量                |
| HGETALL key                                | 返回哈希表key中所有的键值                 |
| HINCRBY  key  field  increment             | 为哈希表key中field键的值加上增量increment |

### 应用场景

#### 对象的缓存

如数据库中的一条数据：有id、name、age、sex等

```
127.0.0.1:6379> hmset people people:id 1 people:name zsl people:age 20
OK
127.0.0.1:6379> hset people people:sex 1
(integer) 1
```

#### 购物

```
--user1的购物车有两个商品
127.0.0.1:6379> hset user1 goods:1 1
(integer) 1
127.0.0.1:6379> hset user1 goods:2 1
(integer) 1
--goods:1买了两个
127.0.0.1:6379> hincrby user1 goods:1 1
(integer) 2
--展示所有的商品
127.0.0.1:6379> hgetall user1
1) "goods:1"
2) "2"
3) "goods:2"
4) "1"
--所有的商品总数
127.0.0.1:6379> hlen user1
(integer) 2
--删除一个商品
127.0.0.1:6379> hdel user1 goods:2
(integer) 1
127.0.0.1:6379> hgetall user1
1) "goods:1"
2) "2"
```

注：过期功能不能使用在field上，只能用在key上。



## 列表list

### 常用操作指令

| 指令                           | 说明                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| LPUSH  key  value [value ...]  | 将一个或多个值value插入到key列表的表头(最左边)               |
| RPUSH  key  value [value ...]  | 将一个或多个值value插入到key列表的表尾(最右边)               |
| LPOP  key                      | 移除并返回key列表的头元素                                    |
| RPOP  key                      | 移除并返回key列表的尾元素                                    |
| LRANGE  key  start  stop       | 返回列表key中指定区间内的元素，区间以偏移量start和stop指定   |
| BLPOP  key  [key ...]  timeout | 从key列表表头弹出一个元素，若列表中没有元素，阻塞等待timeout秒,如果timeout=0,一直阻塞等待 |
| BRPOP  key  [key ...]  timeout | 从key列表表尾弹出一个元素，若列表中没有元素，阻塞等待timeout秒,如果timeout=0,一直阻塞等待 |
| llen key                       | list的长度(数量)                                             |
| lindex key index               | 根据下标获取指定的value                                      |

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1627634671142-18cc19a2-7659-4e34-af0c-dad0e4434b8e.png)

### 应用场景

#### 数据结构

```
--在redis里，先加进入的是尾，后加入的是头
--栈，先进后出
127.0.0.1:6379> lpush stack a b c d
(integer) 4
--移除并返回头元素
127.0.0.1:6379> lpop stack
"d"
127.0.0.1:6379> lpop stack
"c"

--队列，先进先出
127.0.0.1:6379> lpush queue a b c d
(integer) 4
127.0.0.1:6379> rpop queue 
"a"
127.0.0.1:6379> rpop queue 
"b"

--堵塞队列
127.0.0.1:6379> lpush blockqueue a b c d
(integer) 4
127.0.0.1:6379> brpop blockqueue 5
1) "blockqueue"
2) "a"
127.0.0.1:6379> brpop blockqueue 5
1) "blockqueue"
2) "b"
127.0.0.1:6379> brpop blockqueue 0
1) "blockqueue"
2) "c"
127.0.0.1:6379> brpop blockqueue 0
1) "blockqueue"
2) "d"
127.0.0.1:6379> brpop blockqueue 0 //没元素了
..堵塞
```

#### 消息接收

```
--关注的人发送了帖子，打开的时候给用户需要展示近10条帖子
--1.明星1 发了id为01的消息
127.0.0.1:6379> lpush pushMsg:user1 01
(integer) 1
--2.明星2 发了id为02的消息
127.0.0.1:6379> lpush pushMsg:user1 02
(integer) 2
127.0.0.1:6379> lrange pushMsg:user1 0 10
1) "02"
2) "01"
--当前有的帖子数
127.0.0.1:6379> llen pushMsg:user1
(integer) 2
```



## 集合set

### 常用操作指令

| 指令                                     | 说明                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| SADD  key  member  [member ...]          | 往集合key中存入元素，元素存在则忽略，若key不存在则新建       |
| SREM  key  member  [member ...]          | 从集合key中删除元素                                          |
| SMEMBERS  key                            | 获取集合key中所有元素                                        |
| SCARD  key                               | 获取集合key的元素个数                                        |
| SISMEMBER  key  member                   | 判断member元素是否存在于集合key中                            |
| SRANDMEMBER  key  [count]                | 从集合key中选出count个元素，元素不从key中删除                |
| SPOP  key  [count]                       | 从集合key中选出count个元素，元素从key中删除                  |
| 运算指令                                 |                                                              |
| SINTER  key  [key ...]                   | 交集运算                                                     |
| SINTERSTORE  destination  key  [key ..]  | 将交集结果存入新集合destination中                            |
| SUNION  key  [key ..]                    | 并集运算                                                     |
| SUNIONSTORE  destination  key  [key ...] | 将并集结果存入新集合destination中                            |
| SDIFF  key  [key ...]                    | 差集运算(返回第一个集合与其他集合之间的差异，也可以认为说第一个集合中独有的元素) |
| SDIFFSTORE  destination  key  [key ...]  | 将差集结果存入新集合destination中                            |

### 应用场景

#### 抽奖

```
--添加的抽奖用户
127.0.0.1:6379> sadd prize user1 user2 user3 user4
(integer) 4
--查看参与的用户
127.0.0.1:6379> smembers prize
1) "user4"
2) "user2"
3) "user3"
4) "user1"
--随机抽1个人
127.0.0.1:6379> srandmember prize 1
1) "user3"
--抽一个，抽了就没机会再抽别的了
127.0.0.1:6379> spop prize 1
1) "user4"
127.0.0.1:6379> smembers prize
1) "user2"
2) "user3"
3) "user1"
--剩余人数
127.0.0.1:6379> scard prize
(integer) 3
```

#### 集合操作

可以用在共同关注、我的关注、可能认识等。

多级筛选条件等。

```
--三个集合
127.0.0.1:6379> sadd group1 a b c d
(integer) 4
127.0.0.1:6379> sadd group2 b c d
(integer) 3
127.0.0.1:6379> sadd group3 b d e
(integer) 3
--交集
127.0.0.1:6379> sinter group1 group2 group3
1) "d"
2) "b"
--并集
127.0.0.1:6379> sunion group1 group2 group3
1) "c"
2) "e"
3) "a"
4) "d"
5) "b"
--差集
127.0.0.1:6379> sdiff group1 group2 group3
1) "a"
```



## 有序集合zset

### 常用操作指令

| 指令                                      | 说明                                           |
| ----------------------------------------- | ---------------------------------------------- |
| ZADD key score member [[score member]…]   | 往有序集合key中加入带分值元素                  |
| ZREM key member [member …]                | 从有序集合key中删除元素                        |
| ZSCORE key member                         | 返回有序集合key中元素member的分值              |
| ZINCRBY key increment member              | 为有序集合key中元素member的分值加上increment   |
| ZCARD key                                 | 返回有序集合key中元素个数                      |
| ZRANGE key start stop [WITHSCORES]        | 正序获取有序集合key从start下标到stop下标的元素 |
| ZREVRANGE key start stop [WITHSCORES]     | 倒序获取有序集合key从start下标到stop下标的元素 |
| ZUNIONSTORE destkey numkeys key [key ...] | 并集计算                                       |
| ZINTERSTORE destkey numkeys key [key …]   | 交集计算                                       |



### 应用场景

#### 排行榜

```
--点击了一次文章
127.0.0.1:6379>  zincrby news:20210729 1 newsName
"1"
127.0.0.1:6379>  zincrby news:20210729 1 newsName2
"1"
127.0.0.1:6379> zincrby news:20210730 1 newsName
"1"
127.0.0.1:6379> zincrby news:20210730 1 newsName2
"1"
--展示当日排行前五的
127.0.0.1:6379> zrevrange news:20210730 0 4 withscores
1) "newsName2"
2) "1"
3) "newsName"
4) "1"
--近两天newsName的搜索量
127.0.0.1:6379> zunionstore news:20210729-20210730 2 news:20210729 news:20210730
(integer) 2
127.0.0.1:6379> zrange news:20210729-20210730 0 5
1) "newsName"
2) "newsName2"
127.0.0.1:6379> zscore news:20210729-20210730 newsName
"2"
```

# 持久化

即如何保证redis服务挂掉后，重启服务恢复数据。

持久化数据也就是将内存中的数据写入到硬盘里面，大部分原因是为了之后重用数据（比如重启机
器、机器故障之后恢复数据），或者是为了防止系统故障而将数据备份到一个远程位置。

## RDB持久

### 定义

RDB快照(snapshot)。Redis可以通过**创建快照**来获得存储在内存里面的数据在某个时间点上的副本。Redis创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（如主从结构，主要用来提高Redis性能），还可以将快照留在原地以便重启服务器的时候使用。



RDB快照是**默认的持久化方式**， Redis将内存数据库快照保存在名字为 **dump.rdb** 的二进制文件中。 



### 配置说明(自动触发)

```
“N 秒内数据集至少有 M 个改动”这一条件被满足时，自动保存一次数据集。(使用bgsave)
#save N M
以下是redis默认配置的
#save 900 1
#save 300 10
#save 60 10000

#如果是yes，当bgsave命令失败时Redis将停止写入操作。
stop-writes-on-bgsave-error yes

#是否对RDB文件进行压缩
rdbcompression yes

#是否对RDB文件进程校验
rdbchecksum yes

# The filename where to dump the DB 配置文件名称，默认dump.rdb
dbfilename dump.rdb

# 配置rdb文件存放的路径
dir /usr/local/var/db/redis/
```



可以手动执行命令生成RDB快照，进入redis客户端执行命令save或bgsave可以生成dump.rdb文件，每次命令执行都会将所有redis内存快照到一个新的rdb文件里，并覆盖原有rdb快照文件。



### 手动处理(手动触发)

#### save

```
127.0.0.1:6379> save 
OK
```

此命令会使用Redis的主线程进程同步存储，阻塞当前的Redis服务器，造成服务不可用，直到RDB过程完成。

#### bgsave

```
127.0.0.1:6379> bgsave
Background saving started
```

Redis 借助操作系统提供的**写时复制技术(Copy-On-Write, COW)**，在生成快照的同时，依然可以正常处理写命令。简单来说，bgsave 子进程是由**主线程fork**生成的，可以共享主线程的所有内存数据。 

bgsave 子进程运行后，开始读取主线程的内存数据，并把它们写入 RDB 文件。此时，如果主线程对这些数据也都是读操作，那么，主线程和 bgsave 子进程相互不影响。但是，如果主线程要修改一块数据，那么，这块数据就会被复制一份，生成该数据的副本。然后，bgsave 子进程会把这个副本数据写入 RDB 文件，而在这个过程中，主线程仍然可以直接修改原来的数据。



#### 对比

|          | save                 | bgsave                   |
| -------- | -------------------- | ------------------------ |
| IO类型   | 同步                 | 异步                     |
| 是否堵塞 | 是                   | 否                       |
| 复杂度   | O(n)                 | O(n)                     |
| 优点     | 不产生额外内存的消耗 | 不堵塞其它命令           |
| 缺点     | 堵塞其它命令         | 需要fork主线程，占用内存 |



### 定义

只追加文件（ append-only file,AOF ）

RDB方式不能提供强一致性(不耐久)，如果Redis服务因为某些原因崩溃，那么服务器将会丢失最近写入的、没有被保存到rdb文件中去的数据。

AOF的出现很好的解决了数据持久化的实时性，AOF以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中的命令来恢复数据。**AOF会先把命令追加在AOF缓冲区，然后根据对应策略写入硬盘(appendfsync)**。

### 配置说明(自动触发)

```
#是否打开AOF持久化功能
appendonly yes
#AOF文件名称
appendfilename "appendonly.aof"

#同步频率 有三种，默认everysec
#appendfsync always 
appendfsync everysec 
# appendfsync no
#always 	命令写入aof缓冲区后，每一次写入都需要同步，
#					直到写入磁盘（阻塞，系统调用fsync）结束后返回。

#everysec 每秒的命令写入aof缓冲区后，在写入系统缓冲区直接返回（系统调用write），
#					然后有专门线程每秒执行写入磁盘（阻塞，系统调用fsync）后返回。
#					并且在故障时只会丢失 1 秒钟的数据。

#no 			命令写入aof缓冲区后，在写入系统缓冲区直接返回（系统调用write）。
#					之后写入磁盘（阻塞，系统调用fsync）的操作由操作系统负责，通常最长30s。

no-appendfsync-on-rewrite no
#aof文件自上一次重写后文件大小增长了100%则再次触发重写
auto-aof-rewrite-percentage 100
#如果文件大小小于此值不会触发AOF，默认64MB
auto-aof-rewrite-min-size 64mb
```



### 手动处理(手动触发)

```
127.0.0.1:6379> incr readcount
(integer) 1
127.0.0.1:6379> incr readcount
(integer) 2
127.0.0.1:6379> incr readcount
(integer) 3
127.0.0.1:6379> incr readcount
(integer) 4
127.0.0.1:6379> set zsl 666 ex  1000
OK

#对应appendonly.aof 中日志为
*2
$4
incr
$9
readcount
*2
$4
incr
$9
readcount
*2
$4
incr
$9
readcount
*2
$4
incr
$9
readcount
$3
zsl
$3
666
*3
$9
PEXPIREAT
$3
zsl
$13
1627972694321#时间戳
```

注：这是一种resp协议格式数据，*后面的数字代表命令有多少个参数，$号后面的数字代表这个参数有几个字符，如果执行带过期时间的set命令，aof文件里记录的是并不是执行的原始命令，而是记录key过期的时间戳。 



aof手动重写

```
127.0.0.1:6379> bgrewriteaof
Background append only file rewriting started

*2
$4
incr
$9
readcount
4
```

- 多条写入命令可以合并成一条，体积变小

- 重写后AOF文件只保留最终数据的写入命令



在执行 BGREWRITEAOF命令时，Redis 服务器会维护一个 AOF 重写缓冲区，该缓冲区会在子进程创建新AOF文件期间，记录服务器执行的所有写命令。当子进程完成创建新AOF文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新AOF文件的末尾，使得新旧两个AOF文件所保存的数据库状态一致。最后，服务器用新的AOF文件替换旧的AOF文件，以此来完成AOF文件重写操作。

## 对比

|          | RDB          | AOF              |
| -------- | ------------ | ---------------- |
| 文件体积 | 大           | 小               |
| 恢复速度 | 快           | 慢               |
| 启动级别 | 低           | 高               |
| 数据安全 | 容器丢失数据 | 需要看配置的策略 |

## 4.0混合持久

### 定义

工作中一般很少使用RDB模式来恢复内存状态，如上分析，容易丢失大量的数据等。

通常使用AOF方式的日志重放，但是其效率又远比RDB的慢的多，当Redis数据量大的时候，启动花费的时间会特别久。在Redis4.0版本，提供了一个解决方案--混合持久化。



### 配置

```
#开启配置，前提是必须开启aof
aof-use-rdb-preamble yes
```

如果开启了混合持久化。

AOF在重写时，不再是**单纯将内存数据转换为RESP命令写入AOF文件**，而是将重写**这一刻之前的内存做RDB快照**处理，并且将RDB快照内容和增量的AOF修改内存数据的命令**存在一起**，都写入新的AOF文件，新的文件一开始不叫appendonly.aof，等到重写完新的AOF文件才会进行改名，覆盖原有的AOF文件，完成新旧两个AOF文件的替换。 

于是在 Redis 重启的时候，可以先加载 RDB 的内容，然后再重放增量 AOF 日志就可以完全替代之前的 AOF 全量文件重放，因此重启效率大幅得到提升。 

文件内容大致结构：

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1627974601142-8498a046-c935-4e91-90ca-079962c2f660.png)



# 单线程和高性能

## 单线程概念及原因

因为Redis是基于内存的操作，**CPU不是Redis的瓶颈**，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了。

**主要指的是Redis的网络IO和键值对读写是由一个线程完成的。**

**但是其他的功能，如持久化、异步删除等都是额外开辟的线程执行的。**



选用单线程具体原因：

1. **不需要锁来消耗性能**

Redis的数据结构并不全是简单的Key-Value，还有list，hash等复杂的结构，这些结构有可能会进行很细粒度的操作，比如在很长的列表后面添加一个元素，在hash当中添加或者删除一个对象，这些操作可能就需要加非常多的锁，导致的结果是同步开销大大增加。总之，在单线程的情况下，就不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗。

1. **单线程多进程**

单线程的威力实际上非常强大，每核心效率也非常高，多线程自然是可以比单线程有更高的性能上限，但是在今天的计算环境中，即使是单机多线程的上限也往往不能满足需要了，需要进一步摸索的是多服务器集群化的方案，这些方案中多线程的技术照样是用不上的。

1. **CPU消耗**

采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU。

但是如果CPU成为Redis瓶颈，或者不想让服务器其他CUP核闲置，那怎么办？

可以考虑多起几个Redis进程，Redis是key-value数据库，不是关系数据库，数据之间没有约束。只要客户端分清哪些key放在哪个Redis进程上就可以了。



**多进程单线程模型[多进程]**

如下图示：master进程管理worker进程：1. master进程接受到来自外界的信号；2. 向各个worker进程发送信号，并监控worker进程的运行状态；3. 当worker进程退出后【运行结束或异常情况下】，会重新启动新的worker进程。典型代表：Fast-cgi，Nginx，redis, mongodb【写保护，只提供了读同步】。

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1628692962948-112ba861-b1a8-4704-934a-ebe4957f66a9.png)



## 单线程快的原因

Redis所有的数据都存在于**内存中**，于是所有的运算也是**内存级别的运算**，而且单线程的瓶颈不依靠于CPU，避免了多线程的切换导致的性能损耗。

注：Redis是单线程，处理比较快，但是对于耗时的指令需要慎用，一不小心就可能导致Redis卡顿。



## 如何处理多并发(多客户端连接)

查看Redis的连接数

```
# 查看redis支持的最大连接数，在redis.conf文件中可修改 # maxclients 10000 
127.0.0.1:6379> CONFIG GET maxclients
##1) "maxclients"
##2) "10000"
```

### IO多路复用技术

Redis 采用网络IO多路复用技术来保证在多连接的时候， 系统的高吞吐量。

多路-指的是多个socket连接，复用-指的是复用一个线程。

多路复用主要有三种技术：select，poll，epoll。epoll是最新的也是目前最好的多路复用技术。



Redis利用epoll来实现IO多路复用，将连接信息和事件放到队列中，依次放到文件事件分派器，事件分派器将事件分发给事件处理器。 

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1628693841588-9dfc2fb3-1442-4c71-9182-3948b2305c84.png)



# Redis常用架构

## 复制

redis三种主要机制工作：

1. 当主实例和副本实例连接良好时，主实例通过向副本发送命令流来保持副本更新，以复制由于以下原因在主端发生的对数据集的影响：客户端写入、密钥过期或驱逐，任何其他更改主数据集的操作。
2. 当 master 和副本之间的链接中断时，由于网络问题或因为在 master 或副本中检测到超时，副本将重新连接并尝试进行部分重新同步：这意味着它将尝试仅获取部分它在断开连接期间错过的命令流。

1. 当部分重新同步不可能时，副本将要求完全重新同步。这将涉及一个更复杂的过程，其中主节点需要为其所有数据创建快照，将其发送到副本，然后随着数据集的变化继续发送命令流。

Redis 默认使用异步复制，低延迟、高性能，是绝大多数 Redis 用例的自然复制模式。但是，Redis 副本会异步确认它们与主服务器定期收到的数据量。因此，master 不会每次都等待副本处理命令，但是如果需要，它知道哪个副本已经处理了哪个命令。这允许有可选的同步复制。

## 主从架构

### 主从复制

#### 概念

主/从是一种通信模型，其中一个设备或进程对一个或多个其他设备进行单向控制。在某些系统中，从一组符合条件的设备中选出一个主设备，其他设备充当从设备的角色。



主从复制在redis中指的是不同redis服务之间的数据复制，**主-称为主节点(master)，从-称为从节点(slave)**。但是数据复制是单向性的，即只能从节点复制主节点的。redis服务在默认情况下都是主节点。

其特性是：

- 主节点可以没有从节点
- 主节点可以有多个从节点

- 主节点在主从架构中是唯一的，只存在一个



#### 主从优势

**读写均衡**：对于写操作应该去到主节点，对于读操作应该去到从节点。而读操作一般较多，读时从节点的分配由系统指定，如随机法。

**故障恢复**：主写从读，对于从节点来说势必需要去主节点获取最新的数据，且需要定期获取。一般使用asyn方式，相当于从对主中的数据做了备份，当发生故障时，可以通过从来恢复。

#### 主从劣势

**数据不一致性**：从节点复制主节点的过程，必然不可能保证百分之百的成功。比如网络故障、服务宕机等，都可能中断连接。



### 建立主从

**配置文件指定**：

```
相关配置修改：
port 6380 
pidfile /var/run/redis_6380.pid # 把pid进程号写入pidfile配置的文件 
logfile "6380.log" # 指定日志文件
dir /usr/local/redis‐5.0.3/data/6380 # 指定数据存放目录 
# 需要注释掉bind 
#bind 127.0.0.1（bind绑定的是自己机器网卡的ip，如果有多块网卡可以配多个ip，代表允许客户端通过）

配置主从复制： 
replicaof 主节点IP 主节点端口 # 从主节点的redis实例复制数据，Redis 5.0之前使用slaveof
replica‐read‐only yes # 配置从节点只读
masterauth thisreplicationpassword # 主节点密码校验
masteruser replicationuser
```

**server启动指定**：

```
#Redis 5.0之前使用slaveof
./redis-server redis.conf --replicaof 主节点IP 主节点端口 
```

**查看主从状态**：

```
info replication
```



### 主从复制过程

主从复制过程大体可以分为3个阶段：连接建立阶段（即准备阶段）、数据同步阶段、命令传播阶段；下面分别进行介绍。

#### 连接建立过程

[这里实在讲的太清晰了](https://www.cnblogs.com/kismetv/p/9236731.html#t31)

#### 数据同步阶段

主从节点之间的连接建立以后，便可以开始进行数据同步，该阶段可以理解为从节点数据的**初始化**。具体执行的方式是：从节点向主节点发送psync命令（Redis2.8以前是sync命令），开始同步。

而根据主从节点当前状态的不同，可以分为全量复制和部分复制(断点复制)。

#### 命令传播阶段

在此步骤中主节点开始将上一步骤处理的数据(指令)陆续发送给从节点，从节点接收命令并执行，来达到数据的一致性。主节点除了发送复制命令，也会维护心跳机制PING和REPLCONF ACK。

**延迟与不一致**

需要注意的是，命令传播是异步的过程，即主节点发送写命令后**并不会等待从节点的回复**；因此实际上主从节点之间很难保持实时的一致性，延迟在所难免。数据不一致的程度，与主从节点之间的网络状况、主节点写命令的执行频率、以及主节点中的repl-disable-tcp-nodelay配置等有关。



### 工作原理

#### 复制 ID

每个 Redis **主节点**都有一个复制 ID：它是一个**大型伪随机字符串**，用于标记数据集的给定存储。



主从节点初次复制时，主节点将自己的runid发送给从节点，从节点将这个runid保存起来；当断线重连时，从节点会将这个runid发送给主节点；主节点根据runid判断能否进行部分复制：

- 如果从节点保存的runid与主节点现在的runid相同，说明主从节点之前同步过，主节点会继续尝试使用部分复制(到底能不能部分复制还要看offset和复制积压缓冲区的情况)；
- 如果从节点保存的runid与主节点现在的runid不同，说明从节点在断线前同步的Redis节点并不是当前的主节点，只能进行全量复制。



#### offset

每个 master 还采用一个偏移量offset，该偏移量随着它生成的复制流的每个字节而递增，来发送到副本，以便使用修改数据集的新更改来更新副本的状态。即使实际上没有连接副本，复制偏移也会增加，所以基本上每个给定的对：

```
Replication ID, offset
```

#### 复制积压缓冲区

复制积压缓冲区是由主节点维护的、固定长度的、先进先出(FIFO)队列，默认大小1MB(通过配置repl-backlog-size)。当主节点开始有从节点时创建，其作用是备份主节点最近发送给从节点的数据。注意，无论主节点有一个还是多个从节点，都只需要一个复制积压缓冲区。



总结：

当副本连接到主节点时，它们使用**PSYNC**命令来发送它们的旧主副本 ID 和它们到目前为止处理的偏移量。

通过这种方式，master 可以只发送所需的**增量部分**。

从节点将offset发送给主节点后，主节点根据offset和缓冲区大小决定能否执行部分复制：

- 如果offset偏移量之后的数据，仍然都在复制积压缓冲区里，则执行部分复制；
- 如果offset偏移量之后的数据已不在复制积压缓冲区中（数据已被挤出，没有足够的积压），或者保存的master进程ID太旧了，则执行全量复制。



### 数据同步(复制)

#### 主从复制(全量)

(1)如果一个master配置了一个slave节点，不管是否第一次连接上master，其都会发送**psync**命令到master请求复制数据。

(2)当master收到**psync**命令，会进行数据持久化，通过bgsave指令生成rdb快照文件，在此期间，master依然可以接收客户端请求，而且它会把这些请求缓存在**内存**中(**复制缓冲区**)。

(3)当持久化结束后，master会将rdb文件以stream的方式发送给slave，slave接收数据后进行持久化生成rdb，再加载到内存中。然后master再将前面缓存在内存中的命令发送给slave执行，保持一致性。

当然，这里也分为几个场景，如第一次连接肯定需要全量复制；或从节点判断无法进行部分复制，向主节点发送全量复制的请求；或从节点发送部分复制的请求，但主节点判断无法进行部分复制。具体判断就是上述工作原理所述。

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1628953703138-09281588-5f77-45e5-a06a-cb7ae86f0642.png)

如图：主从复制原理图 

#### 部分复制(断点)

当master和slave**断开重连**后，一般都会对整份数据进行复制。

(1)但从redis2.8版本开始，redis改用可以支持部分数据复制的命令PSYNC去master同步数据(**psync <runid> <offset>**)，slave与master能够在网络连接断开重连后只进行**部分数据复制**(**断点续传**)。

(2) master会在其内存中创建一个复制数据用的缓存队列(**复制积压缓冲区**)，缓存最近一段时间的数据，master和它所有的slave都维护了复制的数据下标offset和master的进程id。

(3)因此，当网络连接断开后，slave会请求master 继续进行未完成的复制，从所记录的数据下标开始。如果master进程id变化了，或者从节点数据下标 offset太旧，已经不在master的缓存队列里了，那么将会进行一次全量数据的复制(上述工作原理也有说明)。 	

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1628954887256-00bfe504-83fe-46ab-b59d-50c5386c4b9f.png)

如图：部分复制原理图		

#### psync命令

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1628848828381-494a9e5d-c2bd-46ea-bb63-6678f1a6af10.png)

如图：psync执行图

（1）首先，从节点根据当前状态，决定如何调用psync命令：

- 如果从节点之前未执行过slaveof或最近执行了slaveof no one，则从节点发送命令为psync ? -1，向主节点请求全量复制；
- 如果从节点之前执行了slaveof，则发送命令为psync <runid> <offset>，其中runid为上次复制的主节点的runid，offset为上次复制截止时从节点保存的复制偏移量。

（2）主节点根据收到的psync命令，及当前服务器状态，决定执行全量复制还是部分复制：

- 如果主节点版本低于Redis2.8，则返回-ERR回复，此时从节点重新发送sync命令执行全量复制；
- 如果主节点版本够新，且runid与从节点发送的runid相同，且从节点发送的offset之后的数据在复制积压缓冲区中都存在，则回复+CONTINUE，表示将进行部分复制，从节点等待主节点发送其缺少的数据即可；

- 如果主节点版本够新，但是runid与从节点发送的runid不同，或从节点发送的offset之后的数据已不在复制积压缓冲区中(在队列中被挤出了)，则回复+FULLRESYNC <runid> <offset>，表示要进行全量复制，其中runid表示主节点当前的runid，offset表示主节点当前的offset，从节点保存这两个值，以备使用。

#### 无磁盘化复制

master在内存中直接创建rdb，然后发送给slave，不会在自己本地落地磁盘

通过配置repl-diskless-sync、repl-diskless-sync-delay，等待一定时长再开始复制，因为要等更多slave重新连接过来统一发送。

```tex
repl-diskless-sync no：作用于全量复制阶段，控制主节点是否使用diskless复制（无盘复制）。所谓diskless复制，是指在全量复制时，主节点不再先把数据写入RDB文件，而是直接写入slave的socket中，整个过程中不涉及硬盘；diskless复制在磁盘IO很慢而网速很快时更有优势。需要注意的是，截至Redis3.0，diskless复制处于实验阶段，默认是关闭的。

repl-diskless-sync-delay 5：该配置作用于全量复制阶段，当主节点使用diskless复制时，该配置决定主节点向从节点发送之前停顿的时间，单位是秒；只有当diskless复制打开时有效，默认5s。之所以设置停顿时间，是基于以下两个考虑：(1)向slave的socket的传输一旦开始，新连接的slave只能等待当前数据传输结束，才能开始新的数据传输 (2)多个从节点有较大的概率在短时间内建立主从复制。
```

#### 过期key处理

slave不会过期key，只会等待master过期key。如果master过期了一个key，或者通过LRU淘汰了一个key，那么会模拟一条del命令发送给slave。

###  复制风暴问题

复制风暴是指**多个从节点**对**同一主节点**或者对**同一台机器的多个主节点**短时间内发起全量复制的过程。

复制风暴对发起复制的主节点或者机器造成大量开销，导致 CPU、内存、带宽消耗。

了解了全量复制之后，大概最重要的判断就是从节点缓存的**主节点runid变化**了。顺着这个可分为两种复制场景问题。

- 单节点复制风暴：减少单个主节点关联的从节点
- 单机器复制风暴：分散主节点所在机器部署

#### 单节点下的复制

问题：当master挂了之后重启，runid变化了。会导致下面所有的slave都发生全量复制。

解决：将原来多节点设计，改成树形节点设计(但会增加运维成本)。

**示例图如下**

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1628956175747-270b8007-ac44-41c4-9e32-96ae3247d639.png)



#### 单机器下的复制

问题：如一个机器上部署了多个master，机器挂了之后，会导致各master的所有slave都发生全量复制，而导致带宽不足，无法访问。

解决：避免单台机器上部署过多的主节点，增加分散。

   设定故障迁移机制，如果master所在的机器挂了，即时处理，避免密集的全量复制。

**示例图如下**

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1628956978186-e3d7be87-396a-49eb-8544-b94f51f447b0.png)

### 参考

[https://www.cnblogs.com/kismetv/p/9236731.html](https://www.cnblogs.com/kismetv/p/9236731.html#t21)

## 哨兵架构
