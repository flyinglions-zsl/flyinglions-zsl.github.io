---
title: java-util-stringFormat
date: 2020-05-08 16:44:21
categories:
  - Java-Util
tags:
  - string-format
---

--记录String.format()相关用法

# 使用语法

```java
1.format(String format, Object... args)
```

```java
2.format(Locale l, String format, Object... args)
```



# 类型

| 详细说明 | 示例                                         |                          |
| :------- | :------------------------------------------- | ------------------------ |
| %s       | 字符串类型                                   | “Aa”                     |
| %c       | 字符类型                                     | ‘m’                      |
| %b       | 布尔类型                                     | true                     |
| %d       | 整数类型（十进制）                           | 88                       |
| %x       | 整数类型（十六进制）                         | FF                       |
| %o       | 整数类型（八进制）                           | 77                       |
| %f       | 浮点类型                                     | 8.888                    |
| %.2f     | 浮点类型                                     | 8.88                     |
| %a       | 十六进制浮点类型                             | FF.35AE                  |
| %e       | 指数类型                                     | 9.38e+5                  |
| %g       | 通用浮点类型（f和e类型中较短的）             | 不举例(基本用不到)       |
| %h       | 散列码                                       | 不举例(基本用不到)       |
| %%       | 百分比类型                                   | ％(%特殊字符%%才能显示%) |
| %n       | 换行符                                       | 不举例(基本用不到)       |
| %tx      | 日期与时间类型（x代表不同的日期与时间转换符) | 不举例(基本用不到)       |



# 举例

## %s

```java
System.out.println(String.format("up or %s", "down"));
up or down
```



## %d

```java
System.out.println(String.format("up or down, %d", 888));
up or down, 888
```



## %C

```java
System.out.println(String.format("this is a character, %c", 'a'));
this is a character, a
```



## %b

```java
System.out.println(String.format("boolean is %b", true));
boolean is true
```



## %f

```java
System.out.println(String.format("float number , %f", 8.88888));
float number , 8.888880
```



## %.2f

```java
System.out.println(String.format("float number with precision , %.2f", 8.8888));
float number with precision , 8.89
```



| 标志               | 说明                                                     | 示例                   | 结果             |
| :----------------- | :------------------------------------------------------- | :--------------------- | :--------------- |
| +                  | 为正数或者负数添加符号                                   | (“%+d”,15)             | +15              |
| 0                  | 数字前面补0(加密常用)                                    | (“%04d”, 99)           | 0099             |
| 空格               | 在整数之前添加指定数量的空格                             | (“% 4d”, 99)           | 99               |
| ,                  | 以“,”对数字分组(常用显示金额)                            | (“%,f”, 9999.99)       | 9,999.990000     |
| (                  | 使用括号包含负数                                         | (“%(f”, -99.99)        | (99.990000)      |
| #                  | 如果是浮点数则包含小数点，如果是16进制或8进制则添加0x或0 | (“%#x”, 99)(“%#o”, 99) | 0x63 0143        |
| <                  | 格式化前一个转换符所描述的参数                           | (“%f和%<3.2f”, 99.45)  | 99.450000和99.45 |
| d,%2$s”, 99,”abc”) | 99,abc                                                   |                        |                  |



 %tx x代表日期转换符--日期转换符

| 标志 | 说明                        | 示例                             |
| :--- | :-------------------------- | :------------------------------- |
| c    | 包括全部日期和时间信息      | 星期四 五月 14 22:46:52 CST 2020 |
| F    | “年-月-日”格式              | 2020-05-14                       |
| D    | “月/日/年”格式              | 05/14/20                         |
| r    | “HH:MM:SS PM”格式（12时制） | 10:48:52 下午                    |
| T    | “HH:MM:SS”格式（24时制）    | 22:49:12                         |
| R    | “HH:MM”格式（24时制）       | 22:49                            |



```java
System.out.println(String.format("当前时间： %tc",date));
当前时间： 星期四 五月 14 22:46:52 CST 2020
  
System.out.println(String.format("当前时间： %tF",date));
当前时间： 2020-05-14
  
System.out.println(String.format("当前时间： %tD",date));
当前时间： 05/14/20
 
System.out.println(String.format("当前时间： %tr",date));
当前时间： 10:48:52 下午
  
System.out.println(String.format("当前时间： %tT",date));
当前时间： 22:49:12
  
System.out.println(String.format("当前时间： %tR",date));
当前时间： 22:49
```

