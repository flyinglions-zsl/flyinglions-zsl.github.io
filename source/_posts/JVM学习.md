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

声明周期：

JVM伴随Java程序的开始而开始，程序的结束而停止。一个Java程序会开启一个JVM进程，一台计算机上可以运行多个程序，也就可以运行多个JVM进程。

JVM将线程分为两种：守护线程和普通线程。守护线程是JVM自己使用的线程，比如垃圾回收（GC）就是一个守护线程。普通线程一般是Java程序的线程，只要JVM中有普通线程在执行，那么JVM就不会停止。



启动大概过程：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1616934882154-8796864a-0a25-4773-83da-c87b0436e2b8.png?x-oss-process=image%2Fresize%2Cw_1500)

launcher是一个static且new初始化的变量(即单例)

lancher构造方法会初始化ext类加载器、再由得到的ext类加载器去加载app类加载器。

（app类加载器父类是ext类加载器，ext类加载器原则上是引导类加载器，但引导类是c++创建的，所以其父类加载器为null）  

classLoader->即app类加载器，launcher源码体现。

ClassLoader:

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617460058137-9799f525-5b37-4498-947c-c2080670edcf.png)

## loadClass的类加载过程

**加载(load)>>链接(link)(验证****>>准备****>>解析)****>>初始化****(initialize)****>>使用****>**>卸载

- 加载：在硬盘上查找并通过IO读入字节码文件，使用到类时才会加载，例如调用类的Main()方法，new对象等等，在加载阶段会在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。
- 验证：校验字节码文件的正确性(语义合法、符合逻辑)、文件格式验证(需要符合class文件格式)、元数据验证、符号引用验证(确保解析能够正确执行)等。
- 准备：给类的**静态变量**分配内存，并赋予默认值(如：int的给0，boolean的给false)，final变量直接赋值。这个时候不包括实例变量，实例变量会在对象实例化的时候随着对象一块分配在Java堆中。
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



## Tomcat如何实现打破双亲委派机制

基于自定义的类加载器，是自定义类型的时，不委托父加载器加载。



# JVM内存模型

不同的机器对应的JVM处理机器码的方式都不同，如windows、mac系统

javap -c xx.class看汇编c

## 概念图

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617201607561-5faaec31-3353-4ed4-9c6e-5df99c95ae7e.png)

栈帧：一个方法对应一块栈帧区域，基本就是数据结构中的栈，特性先进后出，后进先出，符合代码执行特性。

操作数栈：临时的内存空间



验证代码：

```
package com.gx.demo.jvm;

public class MyMath {

    public int calculate(int a, int b){
        int result = 0;
        result = (a + b)*5;
        return result;
    }

    public static void main(String[] args) {
        MyMath myMath = new MyMath();
        int res = myMath.calculate(3, 7);
        System.out.println(res);
    }
}
```





## 字节码解析

javap -c MyMath.class

```
{
  public com.gx.demo.jvm.MyMath();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/gx/demo/jvm/MyMath;

  public int calculate(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC
    Code://一个方法对应一个栈帧，即有自己的局部变量表
      stack=2, locals=4, args_size=3//a->局部变量1 b->局部变量2
         0: iconst_0//将int类型常量0压入栈 常量0
         1: istore_3// 将int类型值存入局部变量3 参数result
         2: iload_1//从局部变量1中装载int类型值 参数a
         3: iload_2//从局部变量2中装载int类型值 参数b
         4: iadd//执行int类型的加法
         5: iconst_5//将int类型常量5压入栈 得到a+b=5
         6: imul//乘法
         7: istore_3// 将int类型值存入局部变量3 即5*10=50
         8: iload_3//从局部变量2中装载int类型值 50
         9: ireturn//返回
      LineNumberTable:
        line 6: 0
        line 7: 2
        line 8: 8
      LocalVariableTable://局部变量
        Start  Length  Slot  Name   Signature
            0      10     0  this   Lcom/gx/demo/jvm/MyMath;
            0      10     1     a   I
            0      10     2     b   I
            2       8     3 result   I

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=3, args_size=1
         0: new           #2                  // class com/gx/demo/jvm/MyMath
         3: dup//复制栈顶部一个字长内容
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1//将引用类型或returnAddress类型值存入局部变量1 参数new的myMath，存的堆地址
         8: aload_1//从局部变量1中装载引用类型值
         9: iconst_3//将int类型常量3压入栈
        10: bipush        7   //将一个8位带符号整数压入栈 7 也占一次操作(11)
          //iconst_5 = 8 (0x8)指令码
          //bipush = 16 (0x10)指令码
        12: invokevirtual #4  //调度对象的方法   // Method calculate:(II)I
        15: istore_2//将int类型值存入局部变量2 参数res
        16: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
        19: iload_2//从局部变量2中装载int类型值 calculate返回的结果50
        20: invokevirtual #6                  // Method java/io/PrintStream.println:(I)V
        23: return
      LineNumberTable:
        line 12: 0
        line 13: 8
        line 14: 16
        line 15: 23
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      24     0  args   [Ljava/lang/String;
            8      16     1 myMath   Lcom/gx/demo/jvm/MyMath;
           16       8     2   res   I
}
```

此处涉及一个常量推送到栈顶中，不同范围指令不同：

iconst_*  即-1-5 

bipush    即**-128~127** 

sipush    即-32768~32767 

ldc         即**-2147483648~2147483647**

具体可见官网：https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iconst_i

icons-》push int constant

bipush-》push byte , etc.

| 指令名称    | 指令码（对应class码文件） | 数值 |
| ----------- | ------------------------- | ---- |
| *iconst_m1* | 2 (0x2)                   | -1   |
| *iconst_0 * | 3 (0x3)                   | 0    |
| *iconst_1*  | 4 (0x4)                   | 1    |
| *iconst_2*  | 5 (0x5)                   | 2    |
| *iconst_3*  | 6 (0x6)                   | 3    |
| *iconst_4*  | 7 (0x7)                   | 4    |
| *iconst_5*  | 8 (0x8)                   | 5    |

简单说下calculate方法 运作情况：

1、局部变量和操作数栈各自对应

--》0: iconst_0，0代表程序计数器的位置，每次操作完都会更新

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617203787171-c03c0e63-0f00-4afc-bdeb-9c7cd7408ab5.png)



2、由字节码执行引擎执行操作数为1的步骤,且更新操作数(由字节码执行引擎修改)

--》1: istore_3 程序计数器=1

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617204051112-2efad6e6-54b5-4376-ad5e-7bebf807b85c.png?x-oss-process=image%2Fresize%2Cw_1500)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617203827373-5158c452-ea47-405b-b62d-e4090f98a6c4.png)

3a+b操作

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617204170110-790f84ca-85f2-4e2c-bd35-07841e87b88a.png)

说明：3、7、5等常量都放在操作数栈，临时内存

动态链接：myMath.calculate()，要去解析 找到对应的方法

局部变量和堆：new  MyMath()

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617204681301-ed1af7be-6487-445a-b0ee-271077c47d4f.png)

## 栈

### Java虚拟机栈

Java虚拟机栈是线程私有的，就是平常说的堆栈中的“栈内存”。此处指的是JVM中的运行时内存区域的概念划分，而不是JMM(java 内存模型)，每个方法在执行的同时都会创建一个栈帧(Stack Frame)，方法结束，栈内存也就自动释放了。

其中包括四个部分，分别是：

1局部变量表(用来存储局部变量表的，包含两部分：一是方法中的参数，二是方法中创建的局部变量。局部变量必须被初始化才能使用。包括this，FILO)

2操作数栈(方法用到的常量等，FILO)

3动态链接(方法中对象的引用，存的堆地址等)

4方法出口(方法return到哪个地方)等信息

### 本地方法栈

JVM 调用本地方法(Native)接口时使用，也是线程私有的(历史原因，之前很多接口由C/C++实现的，字节码解析后调用相应的dll文件)，这时候使用的空间就是本地方法栈。

## 程序计数器

在JVM中的运行时内存区域的概念划分中，目前自己了解到主要以下作用，其通过字节码执行引擎工作执行class文件，由字节码执行引擎修改当前这个程序计数器的值来选取下一条需要执行的字节码指令(上面图中有体现)。

(以下为参考)

分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。



JVM的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，为了各条线程之间的切换后计数器能恢复到正确的执行位置，所以每条线程都会有一个独立的程序计数器。



当线程正在执行一个Java方法，程序计数器记录的是正在执行的JVM字节码指令的地址；如果正在执行的是一个Natvie（本地方法），那么这个计数器的值则为空（Underfined）。



程序计数器占用的内存空间很少，也是唯一一个在JVM规范中没有规定任何OutOfMemoryError（内存不足错误）的区域

## 堆

![image.png](https://cdn.nlark.com/yuque/0/2021/png/705191/1617499842167-f422f185-98b9-4655-873e-c3740aa9331b.png)

堆内存是所有线程共享的，基本上可以分为一个年轻代和一个老年代。

新产生的对象都会先放到Eden区(伊甸园)，放满了后进行一次minor gc/gc root(从栈(局部变量)、方法区中去找变量的引用，一直往前找，找到根时如果没有别人对象引用，则会标志为垃圾对象，否则标志为非垃圾对象)。此时非垃圾对象从Eden区转到s0、s1，再依次反复，存活的则保存到老年代(老年代满了则会进行full gc，去掉堆和方法区中的垃圾对象，最终可能报错OOM)。

Stop一the一World，简称STW，指的是Gc事件发生过程中，会产生应用程序的停顿。停顿产生时整个应用程序线程都会被暂停，没有任何响应，有点像卡死的感觉，这个停顿称为STW。



## 方法区

方法区也是所有线程所共享的。它用于存储已经被JVM加载的类信息(类型信息：差不多是C的对象元信息)、常量(存堆中的地址)、静态变量、即时编译器(JIT)编译后的代码等数据信息。且设有保护程序，当一个线程正在访问的时候，另一个线程不能同时加载一个类，需要延迟等待。

同时，方法区中的大小是可以改变的，运行时也可以扩展，对象也可进行垃圾回收，不过条件比较苛刻，需要没有任何引用才会进行回收。

JDK8之前是永久代，之后叫元空间。



# JVM内存参数设置

## 堆

常用：

-Xms：JVM初始分配的内存由-Xms指定，默认是物理内存的1/64

 -Xmx：JVM最大分配的内存由-Xmx指定，默认是物理内存的1/4

新生代： 

-Xmn

-Xmn2G：设置年轻代大小为2G。

## 方法区

-XX:MetaspaceSize 默认21MB(64位JVM)，达到该值则会进行full gc进行类型加载，同时收集器对值进行调整。

-XX:MaxMetaspaceSize 默认无限(64位JVM)，即只限制于本地内存大小

## 栈

-Xss 默认1M，该值设置的越小，说明一个线程栈里面能分配的栈帧就越少，但是对JVM整体来说能开启的线程数就越多。