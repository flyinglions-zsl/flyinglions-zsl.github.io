---
title: redis学习
date: 2021-07-15 22:46:33
tags: 
	- redis
categories:
	- redis

---

基于Redis6.0+版本

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

```shell
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

```shell
127.0.0.1:6379> set name:6 666
OK
127.0.0.1:6379> get name:6
"666"
```

#### 对象的缓存

```shell
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

```shell
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

```shell
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

### 应用场景



## 列表list

### 常用操作指令

### 应用场景



## 集合set

### 常用操作指令

### 应用场景



## 有序集合zset

### 常用操作指令

### 应用场景
