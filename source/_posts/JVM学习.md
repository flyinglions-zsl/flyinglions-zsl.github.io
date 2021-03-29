---
title: JVM学习
date: 2021-03-28 22:59:18
categories:
  - JVM学习
tags:
  - JVM加载过程
---

#  JVM类加载

## JVM整体的加载过程

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1616934882154-8796864a-0a25-4773-83da-c87b0436e2b8.png?x-oss-process=image%2Fresize%2Cw_1500)

launcher是一个static且new初始化的变量(即单例)

lancher构造方法会初始化ext类加载器、再由得到的ext类加载器去加载app类加载器。

（app类加载器父类是ext类加载器，ext类加载器原则上是引导类加载器，但引导类是c++创建的，所以其父类加载器为null）  

classLoader->即app类加载器，launcher源码体现。

## loadClass的类加载过程

**加载>>验证****>>准备****>>解析****>>初始化****>>使用****>**>卸载

- 加载：在硬盘上查找并通过IO读入字节码文件，使用到类时才会加载，例如调用类的Main()方法，new对象等等，在加载阶段会在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。
- 验证：校验字节码文件的正确性。
- 准备：给类的**静态变量**分配内存，并赋予默认值(如：int的给0，boolean的给false)。final变量直接赋值。
- 解析：将符号引用替换为直接引用(内存地址)，该阶段会把一些静态方法（符号引用，比如main()方法）替换为指向数据所存**内存的指针或句柄**等(直接引用)，这就是所谓的**静态链接**过程。**动态链接**是在程序运行期间完成的将符号引用替换为直接引用（每个类名、方法、包路径等都是一个符号 存在于常量池中，运行时加载到运行时常量池中javap xx.class -v 查看）。
- 初始化：对类的静态变量初始化为指定的值，执行静态代码块。 

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1616850631833-55a377a7-b126-40a1-a447-02cec032970e.png?x-oss-process=image%2Fresize%2Cw_1500)

类被加载到方法区中后主要包含 **运行时常量池、类型信息、字段信息、方法信息、类加载器的引用、对应class实例**的引用等信息。

 

## 类加载器

- 引导类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如rt.jar、charsets.jar等
- 扩展类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的ext扩展目录中的JAR类包
- 应用程序类加载器：负责加载CLassPath路径下(项目路径)的类包，主要就是加载自己写的类
- 自定义加载器：负责加载用户自定义路径下的类包



## 双亲委派机制

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1616938423035-31c89842-622b-48db-a80a-20c763c35da5.png)

说明：JVM定义从应用程序类加载器开始加载(具体可看launcher源码 方法getClassLoader())，加载某个类时会委托父加载器寻找目标类，找不到再委托上层父加载器加载，如果所有父加载器都无法在自己的加载类路径下找到目标类，则向下查找并载入目标类。

比如User类，先找到app类加载器加载，app类加载器先委托ext加载器加载，ext加载器再委托引导类加载器加载，引导类加载器在其类加载路径中找不到User类，则向下退回加载User类的请求，ext收到回复就自己加载，ext在其类加载路径中找不到User类，又向下退回给app类加载器，app类加载器则在自己的类加载路径里找User类，找到了则自己加载。95%以上都是app类加载器应用。(源码->appClassLoader的loadClass，跳到super的ClassLoader 的loadClas方法，重点->UrlClassLoader的findCass(name)，装载类即 defineClass() 去类路径拿到class文件)



为什么要设计双亲委派机制？

- 沙箱安全机制：自己的java.lang.String.class类不会被加载，防止核心API库被篡改。

根据双亲委派机制，先会向上加载一次，String类是引导类加载器加载的，这时候引导类加载器找到的是JDK自己的String类且返回(只会加载自己定义的类)，再向下回到app类中，这时候调用则会报错。

- 避免类的重复加载：向上委托，如果父加载器已经加载过了某个类，则会直接返回，子ClassLoader就不会再次加载一次，保证被加载类的唯一性。



## 全盘委托机制

全盘负责 指的是，当一个ClassLoader加载一个类时，除非显示的使用另外一个ClassLoader，该类所依赖及引用的类也由这个ClassLoader载入。（如User类里用到了 Menu类，这时Menu的加载也是由User 的App类加载器加载）



## 自定义类加载器

初始化自定义类加载器，会先初始化父类加载器（默认是app类加载器），其中把自定义类加载器的父类设定为app类加载器，符合双亲委派机制。

1继承ClassLoader

2重写findClass方法

核心就是UrlClassLoader的loadClass()方法(实现双亲委派机制)和findClass()方法(装载类)。



## 打破双亲委派机制

不委托父加载器先加载了，在自定义加载器获得了直接返回。

重写loadClass方法   去掉双亲委派的逻辑