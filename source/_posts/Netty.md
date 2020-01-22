---
title: Hystrix分布式系统限流、降级、熔断框架（二）
date: 2020-01-22 15:33:29
tags: Java
---

------

### Reactor线程模型

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Reactor是反应堆的意思，Reactor模式即Dispatcher模式，服务器程序处理传入的多路请求，将他们同步分派给各请求对应的处理线程。
<br>

Reactor有两个关键角色：
- **Reactor** Reactor在一个单独线程中运行，负责监听和分发事件，将请求事件分发给处理线程来对IO事件作出反应。
- **Handlers** 处理程序执行IO事件完成响应的事件操作。

<br>

![cmd-markdown-logo](http://139.224.113.197/20200121154620.jpg)

根据Reactor和Handler数量的不同，Reactor模型有3个变种。
1. 单Reactor单线程
2. 单Reactor多线程
3. 主从Reactor多线程
主从Reactor包含MainReactor和SubReactor
- MainReactor负责接收请求，并转发给SubReactor
- SubSReactor负责相应通道的读写请求
- 非IO请求的任务直接写入工作队列，并等待Worker Thread处理。

<br>

![cmd-markdown-logo](http://139.224.113.197/20200121155739.jpg)

<br>

### Mina线程模型
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在MINA中，有三种非常重要的线程：Acceptor thread
、Connector thread、I/O processor thread。<br>

**Acceptor thread**：这个线程用于TCP服务器接收新的连接，并将连接分配到I/O processor thread，由I/O processor thread来处理IO操作。每个NioSocketAcceptor创建一个Acceptor thread，线程数量不可配置。

**Connector thread**：用于处理TCP客户端连接到服务器，并将连接分配到I/O processor thread，由I/O processor thread来处理IO操作。每个NioSocketConnector创建一个Connector thread，线程数量不可配置。

**I/O processor thread**：用于处理TCP连接的I/O操作，如read、write。I/O processor thread的线程数量可通过NioSocketAcceptor或NioSocketConnector构造方法来配置，默认是CPU核心数+1。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MINA的TCP服务器包含一个Acceptor thread和多个I/O processor thread，当有新的客户端连接到服务器，首先会由Acceptor thread获取到这个连接，同时将这个连接分配给多个I/O processor thread中的一个线程，当客户端发送数据给服务器，对应的I/O processor thread负责读取这个数据，并执行IoFilterChain中的IoFilter以及IoHandle。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于I/O processor thread本身数量有限，通常就那么几个，但是又要处理成千上万个连接的IO操作，如果有耗时、阻塞的任务，例如查询数据库，那么就会阻塞I/O processor thread，导致无法及时处理其他IO事件，服务器性能下降。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;针对这个问题，MINA中提供了一个ExecutorFilter，用于将需要执行很长时间的会阻塞I/O processor thread的业务逻辑放到另外的线程中，这样就不会阻塞I/O processor thread，不会影响IO操作。

### Netty的线程模型

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Netty是目前流行的由JBOSS提供的一个Java开源框架NIO框架，Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Netty无疑是NIO的老大，它的健壮性、功能、性能、可定制性和可扩展性在同类框架都是首屈一指的。它已经得到成百上千的商业/商用项目验证，如Hadoop的RPC框架Avro、RocketMQ以及主流的分布式通信框架Dubbo等等。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Netty基于JDK自带的NIO的API进行封装，相比JDK原生NIO，Netty提供了相对十分简单易用的API，非常适合网络编程。Netty是完全基于NIO实现的，所以Netty是异步的。
Netty同时是基于主从Reactor多线程模型实现，借鉴了MainReactor和SubReactor结构。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;作为一个异步NIO框架，Netty的所有IO操作都是异步非阻塞的，通过Future-Listener机制，用户可以方便的主动获取或者通过通知机制获得IO操作结果。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Netty的优点可以总结如下：

1. API使用简单，开发门槛低；

2. 功能强大，预置了多种编解码功能，支持多种主流协议；

3. 定制能力强，可以通过ChannelHandler对通信框架进行灵活地扩展；

4. 性能高，通过与其他业界主流的NIO框架对比，Netty的综合性能最优；

5. 成熟、稳定，Netty修复了已经发现的所有JDK NIO BUG，业务开发人员不需要再为NIO的BUG而烦恼；

6. 社区活跃，版本迭代周期短，发现的BUG可以被及时修复，同时，更多的新功能会加入；

7. 经历了大规模的商业应用考验，质量得到验证。在互联网、大数据、网络游戏、企业应用、电信软件等众多行业得到成功商用，证明了它已经完全能够满足不同行业的商业应用了。

Netty服务端的创建方式如下：
```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workGroup = new NioEventLoopGroup();
ServerBootstrap server = new ServerBootstrap();
server.group(bossGroup, bossGroup)
      .channel(NioServerSocketChannel.class);
```
BossGroup和WorkGroup均是线程池

- BossGroup线程池在绑定某个端口后，从BossGroup线程池获取一个线程作为MainReactor后处理端口的accept事件，一个端口对应一个boss。如果只监听一个端口，则BossGroup线程组只创建一个线程即可。
- WorkGroup线程会被SubReactor和worker线程充分利用。

#### Netty的关键组件如下：
- BootStrap、ServerBootStrap <br>
Boostrap客户端启动引导类，ServerBootstrap服务端启动引导类，Bootstrap负责配置整改Netty，并串联起各个组件。

- Channel <br>
Netty网络通信的组件，能够用于执行网络IO操作。

- Selector <br>
Netty基于Selector对象实现I/O多路复用，通过 Selector, 一个线程可以监听多个连接的Channel事件, 当向一个Selector中注册Channel 后，Selector 内部的机制就可以自动不断地查询(select) 这些注册的Channel是否有已就绪的I/O事件(例如可读, 可写, 网络连接完成等)。

- NioEventLoop <br>
NioEventLoop中维护一个线程和任务队列，支持异步提交任务，线程启动时会调用NioEventLoop的run方法，执行IO和非IO任务。 其中IO任务由prossSelectedKeys触发，非IO任务由runAlltasks触发。

- NioEventLoopGroup <br>
NioEventLoopGroup用来管理EventLoop，可以理解为线程组，内部维护了一组线程。  

<br>
Netty的工作架构图如下：


<br>

![cmd-markdown-logo](http://139.224.113.197/20200121161945.jpg)

#### 初始化netty服务端的过程如下
1. 初始化创建2个NioEventLoopGroup，其中boosGroup用于Accetpt连接建立事件并分发请求，workerGroup用于处理I/O读写事件和业务逻辑。
2. 基于ServerBootstrap(服务端启动引导类)，配置EventLoopGroup、Channel类型，连接参数、配置入站、出站事件handler。
3. 绑定端口，开始工作。

server端包含1个Boss NioEventLoopGroup和1个Worker NioEventLoopGroup，NioEventLoopGroup相当于1个事件循环组，这个组里包含多个事件循环NioEventLoop，每个NioEventLoop包含1个selector和1个事件循环线程。

每个Boss NioEventLoop循环执行的任务包含3步
1. 轮询accept事件
2. 处理accept I/O事件，与Client建立连接，生成NioSocketChannel，并将NioSocketChannel注册到某个Worker NioEventLoop的Selector上
3. runAllTasks处理任务队列中的非IO任务。

每个Worker NioEventLoop循环执行的任务包含3步：
1. 轮询read、write事件
2. 处I/O事件，即read、write事件，在NioSocketChannel可读、可写事件发生时进行处理
3. runAllTasks处理任务队列中的非IO任务。

总结：<br>
1. Netty的TCP服务器启动时会创建两个NioEventLoopGroup，一个boss，一个worker。
NioEventLoopGroup实际是是一个线程组，可通过构造函数设置线程数，默认是CPU核数*2。
2. 当有新的TCP连接到服务器时，将有boss线程负责接收，然后将其注册到worder线程，worker线程负责处理IO操作，如read、write。当客户端数据发送到服务端时，worker负责接收数据，并执行ChannelPipeline的ChannelHandler方法。


### Netty和Mina比较
mina与netty都是Trustin Lee的作品，所以在很多方面都十分相似，他们线程模型也是基本一致，采用了Reactors in threads模型，即Main Reactor + Sub Reactors的模式。
1. 都是Trustin Lee的作品，Netty更晚；
2. Mina将内核和一些特性的联系过于紧密，使得用户在不需要这些特性的时候无法脱离，相比下性能会有所下降，Netty解决了这个设计问题；
3. Netty的文档更清晰，很多Mina的特性在Netty里都有；
4. Netty更新周期更短，新版本的发布比较快；
5. 它们的架构差别不大，Mina靠apache生存，而Netty靠jboss，和jboss的结合度非常高，Netty有对google protocal buf的支持，有更完整的ioc容器支持(spring,guice,jbossmc和osgi)；
6. Netty比Mina使用起来更简单，Netty里你可以自定义的处理upstream events或/和downstream events，可以使用decoder和encoder来解码和编码发送内容；
7. Netty和Mina在处理UDP时有一些不同，Netty将UDP无连接的特性暴露出来；而Mina对UDP进行了高级层次的抽象，可以把UDP当成”面向连接”的协议，而要Netty做到这一点比较困难。
8. Netty中的boss线程类似于MINA的Acceptor thread，work线程和MINA的I/O processor thread类似。不同的一点是MINA的Acceptor thread是单个线程，而Netty的boss是一个线程组。