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



# Redis常用架构

## 主从架构



## 哨兵架构
