---
title: Hystrix分布式系统限流、降级、熔断框架
date: 2019-12-12 10:17:09
tags: hystrix
---

------

### 一、为什么要用hystrix

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在大中型分布式系统中，通常系统很多依赖，如下图:

![cmd-markdown-logo](http://139.224.113.197/20191205191911.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在高并发访问下，这些依赖的稳定性与否对系统的影响非常大，但是依赖有很多不可控问题：如网络连接缓慢，资源繁忙，暂时不可用，服务脱机等，如下图：

![cmd-markdown-logo](http://139.224.113.197/20191205191912.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在高流量的情况下，一个后端依赖项的延迟可能导致所有服务器上的所有资源在数秒内饱和（PS：意味着后续再有请求将无法立即提供服务）

![cmd-markdown-logo](http://139.224.113.197/20191205191913.png)

```
例如，对于一个依赖于30个服务的应用程序，每个服务都有99.99%的正常运行时间，期望如下：

0.9999^30  =  99.7% 可用

也就是说一亿个请求的0.3% = 300000 会失败

如果一切正常，那么一年有26.2个小时服务是不可用的

当集群依赖50个服务时：

0.9999^50  =  99.5% 可用， 一年43.8小时不可用
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;分布式系统环境下，服务间类似依赖非常常见，一个业务调用通常依赖多个基础服务。如下图，对于同步调用，当库存服务不可用时，商品服务请求线程被阻塞，当有大批量请求调用库存服务时，最终可能导致整个商品服务资源耗尽，无法继续对外提供服务。并且这种不可用可能沿请求调用链向上传递，这种现象被称为雪崩效应。<br>

![cmd-markdown-logo](http://139.224.113.197/20191205191914.png)

**雪崩效应常见场景**：<br>
**硬件故障：** 如服务器宕机，机房断电，光纤被挖断等；<br>
**流量激增：** 如异常流量，重试加大流量等；<br>
**缓存穿透：** 一般发生在应用重启，所有缓存失效时，以及短时间内大量缓存失效时。大量的缓存不命中，使请求直击后端服务，造成服务提供者超负荷运行，引起服务不可用；<br>
**程序BUG：** 如程序逻辑导致内存泄漏，JVM长时间FullGC等；<br>
**同步等待：** 服务间采用同步调用模式，同步等待造成的资源耗尽。<br>

**雪崩效应应对策略**：<br>
针对造成雪崩效应的不同场景，可以使用不同的应对策略，没有一种通用所有场景的策略，参考如下：<br>
**硬件故障：** 多机房容灾、异地多活等；<br>
**流量激增**：服务自动扩容、流量控制（限流、关闭重试）等；<br>
**缓存穿透：** 缓存预加载、缓存异步加载等；<br>
**程序BUG**：修改程序bug、及时释放资源等；<br>
**同步等待**：资源隔离、MQ解耦、不可用服务调用快速失败等。资源隔离通常指不同服务调用采用不同的线程池；不可用服务调用快速失败一般通过熔断器模式结合超时机制实现。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Hystrix，中文含义是豪猪，因其背上长满棘[jí]刺，从而拥有了自我保护的能力。本文所说的Hystrix是Netflix开源的一款容错框架，同样具有自我保护能力，实现了容错和自我保护。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Netflix Hystrix是SOA/微服务架构中提供服务隔离、熔断、降级机制的工具/框架。Netflix Hystrix是断路器的一种实现，用于高微服务架构的可用性，是防止服务出现雪崩的利器。
#### Hystrix流程解析
![cmd-markdown-logo](http://139.224.113.197/20191205192735.jpg)

**流程说明:**<br>
1. 每次调用创建一个新的HystrixCommand,把依赖调用封装在run()方法中；<br>

2. 执行execute()/queue做同步或异步调用；<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;判断是否使用缓存响应请求，若启用了缓存，且缓存可用，直接使用缓存响应请求。Hystrix支持请求缓存，但需要用户自定义启动；

3. 判断熔断器(circuit-breaker)是否打开,如果打开跳到步骤8,进行降级策略，如果关闭进入步骤4；<br>

4. 判断线程池/队列/信号量是否跑满，如果跑满进入降级步骤8,否则继续后续步骤；<br>

5. 调用HystrixCommand的run方法，运行依赖逻辑：<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5a:依赖逻辑调用超时,进入步骤8；<br>

6. 判断逻辑是否调用成功： <br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6a. 返回成功调用结果<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6b. 调用出错，进入步骤8<br>

7. 计算熔断器状态,所有的运行状态(成功, 失败, 拒绝,超时)上报给熔断器，用于统计从而判断熔断器状态；<br>

8. getFallback()降级逻辑 <br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8a. 没有实现getFallback的Command将直接抛出异常<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8b. fallback降级逻辑调用成功直接返回<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8c. 降级逻辑调用失败抛出异常<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;以下四种情况将触发getFallback调用：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(1). run()方法抛出非HystrixBadRequestException异常。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(2). run()方法调用超时<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(3). 熔断器开启拦截调用<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;(4). 线程池/队列/信号量是否跑满<br>
 
9. 返回执行成功结果。<br>

### 二、Hystrix命令

Hystrix有两个请求命令：HystrixCommand和HystrixObservableCommand。<br>
![cmd-markdown-logo](http://139.224.113.197/20191210164008.jpg)


HystrixCommand用在依赖服务返回单个操作结果的时候。有两种执行方式：

execute()<br>
以同步堵塞方式执行run()，只支持接收一个值对象。hystrix会从线程池中取一个线程来执行run()，并等待返回值。<br>

queue()<br>
以异步非阻塞方式执行run()，只支持接收一个值对象。调用queue()就直接返回一个Future对象。可通过Future.get()拿到run()的返回结果，但Future.get()是阻塞执行的。若执行成功，Future.get()返回单个返回值。当执行失败时，如果没有重写fallback，Future.get()抛出异常。<br>

使用示例：

```
public class MyHystrixCommand extends HystrixCommand<String> {

    private final String param;

    public MyHystrixCommand(String param) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("testGroup")).
                andCommandKey(HystrixCommandKey.Factory.asKey("testCommand")).
                andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("testThreadPool")));
        this.param = param;
    }

    @Override
    protected String run() throws Exception {
        int i = 1/0;    //模拟异常
        return "hello " + param + "!";
    }

    @Override
    protected String getFallback() {
        return "execute faild";
    }

    public static void main(String [] args) throws ExecutionException, InterruptedException {
        MyHystrixCommand command = new MyHystrixCommand("test execute");
        System.out.println(command.execute());

        Future<String> future = new MyHystrixCommand("test queue").queue();
        System.out.println(future.get());
    }
}
```


HystrixObservableCommand 用在依赖服务返回多个操作结果的时候。它也实现了两种执行方式：<br>

observe()<br>
事件注册前执行construct()，支持接收多个值对象，取决于发射源。调用observe()会返回一个hot Observable，也就是说，调用observe()自动触发执行construct()，无论是否存在订阅者。<br>

observe()使用方法：<br>
调用observe()会返回一个Observable对象<br>
调用这个Observable对象的subscribe()方法完成事件注册，从而获取结果。<br>

toObservable()<br>
事件注册后执行construct()，支持接收多个值对象，取决于发射源。调用toObservable()会返回一个cold Observable，也就是说，调用toObservable()不会立即触发执行construct()，必须有订阅者订阅Observable时才会执行。<br>

使用示例：

```
public class MyObservableCommand extends HystrixObservableCommand<String>{

    private final String param;

    public MyObservableCommand(String param) {
        super(HystrixObservableCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("testGroup")).
                andCommandKey(HystrixCommandKey.Factory.asKey("testCommand")));
        this.param = param;
    }

    @Override
    protected Observable<String> construct() {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                try {
                    if(!subscriber.isUnsubscribed()) {
                        subscriber.onNext("Hello");
                        int i = 1 / 2; //模拟异常
                        subscriber.onNext(param + "!");
                        subscriber.onCompleted();
                    }
                } catch (Exception e) {
                    subscriber.onError(e);
                }
            }
        }).subscribeOn(Schedulers.io());
    }

    /**
     * 服务降级
     */
    @Override
    protected Observable<String> resumeWithFallback() {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                try {
                    if (!subscriber.isUnsubscribed()) {
                        subscriber.onNext("execute faild！");
                        subscriber.onNext("figure out the reason！");
                        subscriber.onCompleted();
                    }
                } catch (Exception e) {
                    subscriber.onError(e);
                }
            }
        }).subscribeOn(Schedulers.io());
    }

    public static void main(String [] args){
        Observable<String> observable = new MyObservableCommand("observable").observe();
        Iterator<String> iterator = observable.toBlocking().getIterator();
        while(iterator.hasNext()) {
            System.out.println("observable >>>>" + iterator.next());
        }

        Observable<String> observable2 = new MyObservableCommand("toObservable").toObservable();
        ReplaySubject subject = ReplaySubject.create();
        observable2.subscribe(subject);
        Iterator<String> iterator2 = observable2.toBlocking().getIterator();
        while(iterator2.hasNext()) {
            System.out.println("toObservable >>>>" + iterator2.next());
        }
    }
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果继承的是HystrixCommand，hystrix会从线程池中取一个线程以非阻塞方式执行run()，调用线程不必等待run()；如果继承的是HystrixObservableCommand，将以调用线程堵塞执行construct()，调用线程需等待construct()执行完才能继续往下走。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;需注意的是，HystrixCommand也支持toObservable()和observe()，但是即使将HystrixCommand转换成Observable，它也只能发射一个值对象。只有HystrixObservableCommand才支持发射多个值对象。<br>

几种方法之前的关系<br>

![cmd-markdown-logo](http://139.224.113.197/20191205192741.png)

execute()实际是调用了queue().get()<br>

queue()实际调用了toObservable().toBlocking().toFuture()<br>

observe()实际调用toObservable()获得一个cold Observable，再创建一个ReplaySubject对象订阅Observable，将源Observable转化为hot Observable。因此调用observe()会自动触发执行run()/construct()。<br>

Hystrix总是以Observable的形式作为响应返回，不同执行命令的方法只是进行了相应的转换。<br>