---
title: java-util-stringFormat
date: 2020-05-08 16:44:21
categories:
  - java-util
tags:
  - format
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
| %2f      | 浮点类型                                     | 8.88                     |
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

