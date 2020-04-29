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

1.execute()：以同步堵塞方式执行run()。调用execute()后，hystrix先创建一个新线程运行run()，接着调用程序要在execute()调用处一直堵塞着，直到run()运行完成。

2.queue()：以异步非堵塞方式执行run()。调用queue()就直接返回一个Future对象，同时hystrix创建一个新线程运行run()，调用程序通过Future.get()拿到run()的返回结果，而Future.get()是堵塞执行的。

3.observe()：事件注册前执行run()/construct()。第一步是事件注册前，先调用observe()自动触发执行run()/construct()（如果继承的是HystrixCommand，hystrix将创建新线程非堵塞执行run()；如果继承的是HystrixObservableCommand，将以调用程序线程堵塞执行construct()），第二步是从observe()返回后调用程序调用subscribe()完成事件注册，如果run()/construct()执行成功则触发onNext()和onCompleted()，如果执行异常则触发onError()。

4.toObservable()：事件注册后执行run()/construct()。第一步是事件注册前，调用toObservable()就直接返回一个Observable<String>对象，第二步调用subscribe()完成事件注册后自动触发执行run()/construct()（如果继承的是HystrixCommand，hystrix将创建新线程非堵塞执行run()，调用程序不必等待run()；如果继承的是HystrixObservableCommand，将以调用程序线程堵塞执行construct()，调用程序等待construct()执行完才能继续往下走），如果run()/construct()执行成功则触发onNext()和onCompleted()，如果执行异常则触发onError()
 注：
 execute()和queue()是在HystrixCommand中，observe()和toObservable()是在HystrixObservableCommand 中。从底层实现来讲，HystrixCommand其实也是利用Observable实现的（看Hystrix源码，可以发现里面大量使用了RxJava），尽管它只返回单个结果。HystrixCommand的queue方法实际上是调用了toObservable().toBlocking().toFuture()，而execute方法实际上是调用了queue().get()。

{% asset_img hystrix-xc.jpg this is a jpg aa %}

附：hystrix实现线程隔离：

创建好的线程池是被放入到ConcurrentHashMap中执行依赖代码的线程与请求线程(比如Tomcat线程)分离，请求线程可以自由控制离开的时间，这也是我们通常说的异步编程，Hystrix是结合RxJava来实现的异步编程。通过设置线程池大小来控制并发访问量，当线程饱和的时候可以拒绝服务，防止依赖问题扩散

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

```
super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("xxxCommand"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                				//采样时间间隔 default 500
                				.withMetricsHealthSnapshotIntervalInMilliseconds(3000)
                	      //是否开启熔断，默认true
                        .withCircuitBreakerEnabled(true)
                        //请求量阈值：将使断路器打开的滑动窗口中的最小请求数量。默认20；例如，如果值是20，那么如果在滑动窗口中只接收到19个请求(比如一个10秒的窗口)，即使所有19个请求都失败了，断路器也不会打开。
                        .withCircuitBreakerRequestVolumeThreshold(10)
                        //断路器打开后拒绝请求的时间量，即工作时间。默认5000ms,熔断器中断请求5秒后会关闭重试,如果请求仍然失败,继续打开熔断器5秒,如此循环
                        .withCircuitBreakerSleepWindowInMilliseconds(5000)
                        //错误百分比阈值；当请求错误率超过设定值，断路器就会打开，默认50，单位%
                        .withCircuitBreakerErrorThresholdPercentage(50)
                        )); 
```

### 线程池相关

```
super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("xxxCommand"))
                .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
                		//核心线程数;Hystrix使用的是JUC线程池ThreadPoolExecutor，线程池相关配置直接影响ThreadPoolExecutor实例;默认10
                        .withCoreSize(10)
                        //最大线程数;默认10
                        .withMaximumSize(10)
                        //最大任务队列容量;默认-1;此属性配置为-1时使用的是SynchronousQueue，配置为大于1的整数时使用的是LinkedBlockingQueue;allowMaximumSizeToDivergeFromCoreSize为true的时候才生效
                        .withMaxQueueSize(-1)
                        //任务拒绝的任务队列阈值;默认5;maxQueueSize配置为-1的时候，此配置项不生效
                        .withQueueSizeRejectionThreshold(5)
                        //非核心线程存活时间;allowMaximumSizeToDivergeFromCoreSize为true并且maximumSize大于coreSize时此配置才生效;默认1分钟
                        .withKeepAliveTimeMinutes(1)
                        //是否允许最大线程数生效;默认false
                        .withAllowMaximumSizeToDivergeFromCoreSize(true)
                        ));
```

参数配置参考博客：

https://www.cnblogs.com/throwable/p/11961016.html

https://www.cnblogs.com/chongaizhen/p/11132066.html

# 四种触发fallback的情况

**1.**非**HystrixBadRequestException**异常：当抛出**HystrixBadRequestException**时，调用程序可以捕获异常，此时不会触发**fallback**，而其他异常则会触发**fallback**，调用程序将获得**fallback**逻辑的返回结果。

**2.run**()**/construct**()运行超时：执行命令的方法超时，将会触发**fallback**。

**3.**熔断器开启：当熔断器处于开启的状态，将会触发**fallback**。

**4.**线程池**/**信号量已满：当线程池**/**信号量已满的状态，将会触发**fallback**。

# fallback

```
super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("xxxCommand"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                //最大并发降级请求处理上限;getFallback()方法在执行线程中调用的最大上限，如果超过此上限，降级逻辑不会执行并且会抛出一个异常，默认10，
                        .withFallbackIsolationSemaphoreMaxConcurrentRequests(20)
                        //是否开启降级;默认true
                        .withFallbackEnabled(true)
                        ));
```

fallback也可实现多级跳转：

HystrixCommand1执行fallback1， fallback1的执行嵌入HystrixCommand2,当HystrixCommand2执行失败的时候，触发HystrixCommand2的fallback2，以此循环下去实现多级fallback

{% asset_img hyxtrix-fallback.png this is a jpg bb%}

# 总结

大概了解了hystrix其基本的原理、应用以及各种参数配置等。需要注意的是使用注解时，加了注解的方法就是被监控的方法，而自定义时，执行的run方法需要人为的放入其中：

```
@Autowired
OrderService orderService;
 @Override
    protected String run() throws Exception {
    		orderService.getSomething();
        return null;
    }
```

麻烦的是run不能传参，酌情而用。

