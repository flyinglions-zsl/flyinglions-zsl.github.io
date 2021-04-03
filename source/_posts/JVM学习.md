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

根据验证代码 走一遍逻辑

1代码

2查指令 百度网盘下载

3画栈中局部变量和操作数栈 变化的图逻辑

4堆与方法区的关系

5堆与栈的关系

6程序计数器 变化，具体xx.class由字节码执行引擎执行



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