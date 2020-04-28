---
title: hystrix原理及应用
categories:
  - SpringCloud
tags:
  - hystrix
---

前言：近来接触hystrix，需要给接口进行熔断处理，记录下应用及学习。

# 初识hystrix

## hystrix原理

上原理图：

{% asset_img hystrix-yuanli.jpg this is a jpg %}

Hystrix整个工作流如下：

1. 构造一个 HystrixCommand或HystrixObservableCommand对象，用于封装请求，并在构造方法配置请求被执行需要的参数；
2. 执行命令，Hystrix提供了4种执行命令的方法，后面详述；
3. 判断是否使用缓存响应请求，若启用了缓存，且缓存可用，直接使用缓存响应请求。Hystrix支持请求缓存，但需要用户自定义启动；
4. 判断熔断器是否打开，如果打开，跳到第8步；
5. 判断线程池/队列/信号量是否已满，已满则跳到第8步；
6. 执行HystrixObservableCommand.construct()或HystrixCommand.run()，如果执行失败或者超时，跳到第8步；否则，跳到第9步；
7. 统计熔断器监控指标；
8. 走Fallback备用逻辑
9. 返回请求响应

## 4种执行方法

- `execute()`—该方法是阻塞的，从依赖请求中接收到单个响应（或者出错时抛出异常）。
- `queue()`—从依赖请求中返回一个包含单个响应的Future对象。
- `observe()`—订阅一个从依赖请求中返回的代表响应的Observable对象。
- `toObservable()`—返回一个Observable对象，只有当你订阅它时，它才会执行Hystrix命令并发射响应。

# hystrix熔断实现手段

## 注解

通过在目标方法（public）上添加 @HystrixCommand，可以指定CommandKey来在配置文件当中单独的对某个command进行配置。

```java
@HystrixCommand(commandKey="",groupKey="",fallback="",
                commandProperties={@HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE")})
```

## 自定义HystrixCommand

通过继承HystrixCommand<?> 覆写run、getfallback方法，?代表run方法返回类型。

```java
public class CustomCommand extends HystrixCommand<String> {

    public CustomCommand() {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("CustomCommand"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                        .withExecutionTimeoutEnabled(true)));
    }

    @Override
    protected String run() throws Exception {
        return null;
    }
    
    @Override
    protected String getFallback() {
        return "";
    }
}
```

## 配置参数简介

### HystrixCommandGroupKey

用于对`Hystrix`命令进行分组，分组之后便于统计展示于仪表盘、上传报告和预警等等，也就是说，`HystrixCommandGroupKey`是`Hystrix`内部进行度量统计时候的分组标识，数据上报和统计的最小维度就是分组的KEY。`HystrixCommandGroupKey`是必须配置的。

```
HystrixCommandGroupKey.Factory.asKey("Group Key")
public Command() {
    super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("Group Key")));
}
```

### HystrixCommandKey

它是`Hystrix`命令的唯一标识，准确来说是`HystrixCommand`实例或者`HystrixObservableCommand`实例的唯一标识。

```
HystrixCommandKey.Factory.asKey("Your Key");
public Command() {
    super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("Group Key"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("Command Key")));
}
```

### HystrixThreadPoolKey

一个`HystrixCommand`会和一个独立的`HystrixThreadPool`实例关联，用于监控、度量和缓存等（拒绝）。如果没有配置，默认线程池名会使用group Key作为名称。

```
HystrixThreadPoolKey.Factory.asKey("ThreadPoolKey")
public Command() {
    super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("xxx"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("YYY"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("ThreadPoolKey")));
}
```

### 超时相关

（默认true，1000ms，单位：ms）

```
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds

super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("xxxCommand"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                        .withExecutionTimeoutEnabled(true) //是否开启
                        .withExecutionTimeoutInMilliseconds(1000) //超时时间
                        .withExecutionIsolationThreadInterruptOnTimeout(true)//超时是否中断
                        );
```

### 基本并发相关

```
super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("xxxCommand"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()  
                //线程拒绝策略（SEMAPHORE和THREAD）
                .withExecutionIsolationStrategy(
                      HystrixCommandProperties.ExecutionIsolationStrategy.SEMAPHORE)
                 //最大并发请求数，默认10，
                 .withExecutionIsolationSemaphoreMaxConcurrentRequests(100)
                 //最大并发降级请求处理上限;getFallback()方法在执行线程中调用的最大上限，如果超过此上限，降级逻辑不会执行并且会抛出一个异常，默认10，
                 .withFallbackIsolationSemaphoreMaxConcurrentRequests(20)
                 );
```

### 熔断相关

