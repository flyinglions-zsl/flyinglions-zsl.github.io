---
title: spring源码01-浅识
date: 2021-01-20 19:46:52
categories:
  - spring源码
tags:
  - spring源码
---

# 1.怎么学习源码

## 1.1编译

使用gradle编译spring.5.0.x版本

坑点：由于spring官方有自己的gradle仓库，下载较慢，需要配置阿里镜像

常见问题：

1.找不到spring-beans 的依赖。注释spring-beans.gradle 29行

2.idea2019指定本地安装的gradle版本。preferences->grade->user grade from 选择 specified location 指定本地gradle home路径

## 1.2思路

猜想 + 验证

实现基本的springmvc执行流程(带web.xml的)

### 1.2.1配置

| 配置web.xml （DispatcherServlet）                            |
| ------------------------------------------------------------ |
| 配置init-param                                               |
| 配置url-pattern                                              |
| 仿照spring 配置同套的annotation 如 @Controller @Service @Autowired @RequestMapping ... |

### 1.2.2初始化

| 调用init()方法 加载配置文件                         |
| --------------------------------------------------- |
| IOC容器初始化                                       |
| 扫描相关类 scan-package                             |
| 实例化且加入Spring容器(反射机制)                    |
| DI 操作，扫描容器中的实例，给没有赋值的属性自动赋值 |
| 初始化handlerMapping 等 url和method进行关联映射     |

#### 1.2.3运行

| 调用doPost/doGet方法 获取request/response对象  |
| ---------------------------------------------- |
| 匹配handlerMapping                             |
| 反射调用method.invoke()方法 执行相关逻辑       |
| response.getWrite().write() 将结果返回给浏览器 |

