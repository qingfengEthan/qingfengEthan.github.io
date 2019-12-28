---
title: Hystrix分布式系统限流、降级、熔断框架（二）
date: 2019-12-29 14:37:57
tags: hystrix
---

------
### 三、Hystrix容错
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Hystrix的容错主要是通过添加容许延迟和容错方法，帮助控制这些分布式服务之间的交互。 还通过隔离服务之间的访问点，阻止它们之间的级联故障以及提供回退选项来实现这一点，从而提高系统的整体弹性。Hystrix主要提供了以下几种容错方法：<br>

- **资源隔离**
- **熔断**
- **降级**

#### 1、资源隔离-线程池 
<br>

![cmd-markdown-logo](http://139.224.113.197/20191220155588.png)

![cmd-markdown-logo](http://139.224.113.197/20191220155589.png)

<br>

<br>

#### 2、资源隔离-信号量
<br>

![cmd-markdown-logo](http://139.224.113.197/20191205192739.jpg)
<br>
<br>

![cmd-markdown-logo](http://139.224.113.197/20191205192740.png)

##### 线程池和信号量隔离比较

&nbsp; | 线程切换 | 支持异步 | 支持超时 | 支持熔断 | 限流 | 开销
---|---|---|---|---|---|---|---
信号量| 否 | 否 | 否 | 是 | 是 | 小
线程池| 是 | 是 | 是 | 是 | 是 | 大


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当请求的服务网络开销比较大的时候，或者是请求比较耗时的时候，我们最好是使用线程隔离策略，这样的话，可以保证大量的容器(tomcat)线程可用，不会由于服务原因，一直处于阻塞或等待状态，快速失败返回。而当我们请求缓存这些服务的时候，我们可以使用信号量隔离策略，因为这类服务的返回通常会非常的快，不会占用容器线程太长时间，而且也减少了线程切换的一些开销，提高了缓存服务的效率。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;线程池适合绝大多数的场景，对依赖服务的网络请求的调用；信号量适合访问不是对外部依赖的访问，而是对内部的一些比较复杂的业务逻辑的访问，只要做信号量的普通限流就可以了。<br>

#### 3、熔断
**为什么要使用断路器**？<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在分布式架构中，一个应用依赖多个服务是非常常见的，如果其中一个依赖由于延迟过高发生阻塞，调用该依赖服务的线程就会阻塞，如果相关业务的QPS较高，就可能产生大量阻塞，从而导致该应用/服务由于服务器资源被耗尽而拖垮。另外，故障也会在应用之间传递，如果故障服务的上游依赖较多，可能会引起服务的雪崩效应。

![cmd-markdown-logo](http://139.224.113.197/20191209165701.png)

熔断器工作的详细过程如下：

第一步，调用allowRequest()判断是否允许将请求提交到线程池

1、如果熔断器强制打开，circuitBreaker.forceOpen为true，不允许放行，返回。<br>
2、如果熔断器强制关闭，circuitBreaker.forceClosed为true，允许放行。此外不必关注熔断器实际状态，也就是说熔断器仍然会维护统计数据和开关状态，只是不生效而已。

第二步，调用isOpen()判断熔断器开关是否打开

1、如果熔断器开关打开，进入第三步，否则继续；<br>
2、如果一个周期内总的请求数小于circuitBreaker.requestVolumeThreshold的值，允许请求放行，否则继续；<br>
3、如果一个周期内错误率小于circuitBreaker.errorThresholdPercentage的值，允许请求放行。否则，打开熔断器开关，进入第三步。

第三步，调用allowSingleTest()判断是否允许单个请求通行，检查依赖服务是否恢复

1、如果熔断器打开，且距离熔断器打开的时间或上一次试探请求放行的时间超过circuitBreaker.sleepWindowInMilliseconds的值时，熔断器器进入半开状态，允许放行一个试探请求；否则，不允许放行。<br>

此外，为了提供决策依据，每个熔断器默认维护了10个bucket，每秒一个bucket，当新的bucket被创建时，最旧的bucket会被抛弃。其中每个blucket维护了请求成功、失败、超时、拒绝的计数器，Hystrix负责收集并统计这些计数器。

![cmd-markdown-logo](http://139.224.113.197/20191205192736.png)

执行策略如下：

Hystrix遇到一个超时/失败请求，此时启动一个10s的窗口，后续的请求会进行如下判断：

（1）查看失败次数是否超过最小调用次数

- 如果没有超过，则放行请求。
- 如果超过最小请求数，继续下面逻辑

（2）判断失败率是否超过一个阈值，这里错误是指超时和失败两种。

- 如果没有超过，则放行
- 如果超过错误阈值，则继续下面逻辑

（3）熔断器断开

- 请求会直接返回失败。
- 会开一个5s的窗口，每隔5s调用一次请求，如果成功，表示下游服务恢复，否则继续保持断路器断开状态。


#### 4、降级
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;降级，通常指务高峰期，为了保证核心服务正常运行，需要停掉一些不太重要的业务，或者某些服务不可用时，执行备用逻辑从故障服务中快速失败或快速返回，以保障主体业务不受影响。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;要支持回退或降级处理，一般是查询操作，可以重写HystrixCommand的getFallBack方法或HystrixObservableCommand的resumeWithFallback方法，通常不建议在回退逻辑中执行任何可能失败的操作。<br>
Hystrix在以下几种情况下会走降级逻辑：

- 执行construct()或run()抛出异常<br>
- 熔断器打开导致命令短路<br>
- 命令的线程池和队列或信号量的容量超额，命令被拒绝<br>
- 命令执行超时

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果降级逻辑中需要发起远程调用，建议重新封装一个HystrixCommand，使用不同的ThreadPoolKey，与主线程池进行隔离。

### 四、Hystrix配置
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Hystrix默认使用Netflix Archaius进行配置管理，项目中使用zookeeper作为配置源，通过archaius-zookeeper实现hystrix命令、熔断器、线程池、监控等参数的动态配置，根据生产环境需要动态调整hystrix参数，实现了对微服务的治理。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每个Hystrix参数都有4个地方可以配置，优先级从低到高如下，如果每个地方都配置相同的属性，则优先级高的值会覆盖优先级低的值：
- 内置全局默认值：写死在Hystrix代码里的默认值,如HystrixCommandProperties.default_executionTimeoutInMilliseconds属性
- 动态全局默认属性：全局配置文件读到的默认值
- 内置实例默认值：创建HystrixCommand时，通过注解或者给父类构造器传参的方式设置的默认值
- 动态配置实例属性：通过属性文件配置特定实例的值

以hystrix命令sns.grassSearchIndex执行的超时时间设置为例，属性的优先级从低到高为

```
default_executionTimeoutInMilliseconds=1000
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=500
HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(1000)
hystrix.command.sns.grassSearchIndex.execution.isolation.thread.timeoutInMilliseconds=1500
```



#### Command 配置

```
# 隔离策略: 可选THREAD｜SEMAPHORE，默认TREAD
hystrix.command.default.execution.isolation.strategy=TREAD
# 服务超时时间，单位毫秒，默认1000
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=3000 
# 服务sns.grassSearchIndex超时时间
hystrix.command.sns.grassSearchIndex.execution.isolation.thread.timeoutInMilliseconds=2000
# 是否打开超时检测，默认启用true
hystrix.command.default.execution.timeout.enabled=true
# 使用信号量隔离时qps阈值，后续的请求会被拒，默认10
hystrix.command.default.execution.isolation.semaphore.maxConcurrentRequests=10
```


#### 熔断器（Circuit Breaker）配置

```
# 是否打开断路器，默认开启true
hystrix.command.default.circuitBreaker.enabled=true
# 断路器检测的基础请求值，只有时间窗口内的请求数达到这个阈值时，才会判定错误率，否则比如只有一两个请求，即便都失败了，也不会打开断路器，因为基数太少了，默认20
hystrix.command.default.circuitBreaker.requestVolumeThreshold=100
# 错误百分比，超过就会短路，默认值50
hystrix.command.default.circuitBreaker.errorThresholdPercentage=75
# 指的是从断路器打开状态到半开半闭状态需要的时间，即断路后，需要等多久才能放一个请求进来，默认值5000
hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds=5000

# Metrics
# 统计的时间窗口大小，默认10000
hystrix.command.default.metrics.rollingStats.timeInMilliseconds=10000
# 时间窗口的桶的数目，必须能被时间窗口大小整除，否则报错，每个bucket包含success，failure，timeout，rejection的次数的统计信息，默认10
hystrix.command.default.metrics.rollingStats.numBuckets=10
# 每一次检测的间隙。因为就算分窗口统计错误率，也会很占cpu，所以每一次统计都会等一个时间间隔再开始，默认500
hystrix.command.default.metrics.healthSnapshot.intervalInMilliseconds=500
# 带rollingPercentile的都表示调用时延的统计，该选项表示是否打开时延统计，比如说95分位99分位等，如果关闭都返回-1，默认true
hystrix.command.default.metrics.rollingPercentile.enabled=true
# 时延统计的时间窗口,默认60000
hystrix.command.default.metrics.rollingPercentile.timeInMilliseconds=60000 
# 时延统计的桶数目
hystrix.command.default.metrics.rollingPercentile.numBuckets=6
# 时延统计的桶大小，时延统计每一个桶只维持最新的该数值的请求的数据，早一些的将会被覆盖。如果bucket size＝100，window＝10s，若这10s里有500次执行，只有最后100次执行会被统计到bucket里去，增加该值会增加内存开销以及排序的开销，默认100
hystrix.command.default.metrics.rollingPercentile.bucketSize=100 

```


#### 线程池(ThreadPool)配置

```
# 默认核心线程数，不会变，默认10
hystrix.threadpool.default.coreSize=30
# userLogin隔离线程池核心线程数
hystrix.threadpool.userLogin.coreSize=20
# 等待队列，还超就会被拒，这个数值无法动态修改，默认-1
hystrix.threadpool.default.maxQueueSize=50000
# userLogin隔离线程池等待队列
hystrix.threadpool.userLogin.maxQueueSize=3000
# 进入queue时被拒的概率值，即便是没有达到maxQueueSize。这个为了弥补上面无法动态修改的不足。可以通过这个概率值来控制队列大小
hystrix.threadpool.default.queueSizeRejectionThreshold=45000
#线程池统计指标的时间，默认10000
hystrix.threadpool.default.metrics.rollingStats.timeInMilliseconds=10000
#将rolling window划分为n个buckets，默认10
hystrix.threadpool.default.metrics.rollingStats.numBuckets=10

```



### 五、服务监控

##### Hystrix Dashboard
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Hystrix Dashboard主要用来实时监控Hystrix的各项指标信息。通过Hystrix Dashboard反馈的实时信息，可以帮助我们快速发现系统中存在的问题。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过 [https://search.maven.org](https://search.maven.org) 站点下载standalone-hystrix-dashboard，运行jar包后访问http://localhost:7979/hystrix-dashboard/，即可进入hystrix dashboard页面
```
nohup java -jar -DserverPort=7979 -DbindAddress=localhost standalone-hystrix-dashboard-1.5.3-all.jar &
```
![cmd-markdown-logo](http://139.224.113.197/20191210104701.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;集群环境监控可使用Netflix提供的turbine进行监控。通过maven公服https://search.maven.org下载并部署war包turbine-web，修改集群节点配置，将turbine地址http://localhost:${port}/turbine.stream?cluster=default添加监控到dashboard 

![cmd-markdown-logo](http://139.224.113.197/20191219150201.png)

```
turbine.aggregator.clusterConfig=default
turbine.instanceUrlSuffix=:8080/gateway/hystrix.stream
turbine.ConfigPropertyBasedDiscovery.test.instances=10.66.70.1,10.66.70.2,10.66.70.3
```