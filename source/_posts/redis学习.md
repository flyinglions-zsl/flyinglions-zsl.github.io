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
