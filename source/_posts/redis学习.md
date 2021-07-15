---
title: redis学习
date: 2021-07-15 22:46:33
tags: 
	- redis
categories:
	- redis

---

基于Redis6.5版本

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



### 应用场景

#### 单个值的缓存

#### 对象的缓存

#### 计数器

#### 分布式锁





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
