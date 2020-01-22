---
title: BIO、NIO、AIO模型分析
date: 2020-01-17 09:39:19
tags: Java
---

------
### 什么是 I/O
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在计算机系统中I/O就是输入（Input）和输出(Output)的意思，针对不同的操作对象，可以划分为磁盘I/O模型，网络I/O模型，内存映射I/O, Direct I/O、数据库I/O等，常见的I/O有磁盘I/O和网络I/O。
<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;什么是IO的Block呢？考虑下面两种情况：
- 用系统调用read从socket里读取一段数据
- 用系统调用read从一个磁盘文件读取一段数据到内存

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对于第一种情况，算作Block，因为Linux无法知道网络上对方是否会发数据。如果没数据发过来，对于调用read的程序来说，就只能“等”。对于第二种情况，不算做Block。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个解释是，所谓“Block”是指操作系统可以预见这个Block会发生才会主动Block。例如当读取TCP连接的数据时，如果发现Socket buffer里没有数据就可以确定定对方还没有发过来，于是Block；而对于普通磁盘文件的读写，也许磁盘运作期间会抖动，会短暂暂停，但是操作系统无法预见这种情况，只能视作不会Block，照样执行。我们讨论的BIO、NIO和AIO都是针对网络I/O模型。

进程中的IO调用步骤大致可以分为以下四步： 

1. 进程向操作系统请求数据 ;
2. 操作系统把外部数据加载到内核的缓冲区中; 
3. 操作系统把内核的缓冲区拷贝到进程的缓冲区 ;
4. 进程获得数据完成自己的功能 ;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当操作系统在把外部数据放到进程缓冲区的这段时间（即上述的第2，3步）可以分为两个阶段：OS把外部数据加载到内核的缓冲区和OS把内核的缓冲区拷贝到进程的缓冲区。根据这两个阶段的情况产生了几种不同的IO模型：BIO、NIO、IO多路复用和AIO，如果应用进程是挂起等待的，那么就是同步IO，反之，就是异步IO，也就是AIO 。

### 同步与异步
- **同步** 是指发出一个请求，在没有得到结果之前该请求就不返回结果。请求不返回结果之前不去处理其他请求。<br>
- **异步** 是指发出一个请求后，立刻得到了回应，但没有返回结果。这时我们可以再处理别的事情(发送其他请求)，所以这种方式需要我们通过状态主动查看是否有了结果, 或者可以设置一个回调来通知调用者。
- **同步与异步** 关注的是 **消息通知机制** 。
- 同步做一件事情的时候，不能做其他事情，或者不停的去问询事情的处理结果。<br>  异步可以去做其他事情，等待事情结束之后通知，而不需要一直等待或者不时的去问。<br>


### 阻塞与非阻塞
- **阻塞** 是指请求结果返回之前，当前线程会被挂起(被阻塞)，这时线程什么也做不了。<br>
- **非阻塞** 是指请求结果返回之前，当前线程没有被阻塞，仍然可以做其他事情。
- **阻塞和非阻塞** 关注的是 **程序等待调用结果时的状态**。<br>
- **阻塞与非阻塞** 关注的是对IO操作的不同的方式，阻塞方式下，必须等到读写完成才返回，非阻塞状态下，可以立即返回，等到读写完成之后才告知客户端已经处理完毕。


### BIO（同步阻塞）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;BIO即Blocking IO，JDK1.4 版本之前，消息之间通信使用的是BIO模式，服务端启动一个socket连接，当有连接请求过来的时候，服务端创建线程，或者从线程池中获取线程来处理，当服务端的线程满了之后，就无法继续处理。要求客户端等待，或者直接拒绝。等到服务端释放线程处理。
<br>

![cmd-markdown-logo](http://139.224.113.197/20200120175011.jpg)
<br>
- BIO弊端：线程资源是宝贵且有限，大量的线程会占用大量的内存空间，同时线程间的上下文切换也会消耗很多资源。
- BIO的特点就是在IO执行的两个阶段都被block了。
- BIO核心是：**一个连接一个线程**。

<br>

![cmd-markdown-logo](http://139.224.113.197/20200122101706.png)
<br>
为避免请求创建的连接不做任何事情而造成不必要的线程开销，BIO可以通过线程池机制改善 (伪NIO）。
<br>

![cmd-markdown-logo](http://139.224.113.197/20200120175012.jpg)
<br>

### NIO（同步非阻塞）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NIO即Non-blocking IO，同步非阻塞，服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。<br>
NIO中轮询操作是用户线程进行的<br>

- NIO的特点就是程序需要不断的主动询问内核数据是否准备好。第一个阶段非阻塞，第二个阶段阻塞。
- NIO核心是 **一个请求一个线程** 。
- tomcat 6之后开始支持使用NIO模型。
<br>

![cmd-markdown-logo](http://139.224.113.197/20200122103509.jpg)
<br>
####  缓冲区Buffer
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在NIO库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的；在写入数据时，也是写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作。
<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;缓冲区实际上是一个数组，并提供了对数据结构化访问以及维护读写位置等信息。具体的缓存区有这些：ByteBuffe、CharBuffer、 ShortBuffer、IntBuffer、LongBuffer、FloatBuffer、DoubleBuffer。他们实现了相同的接口：Buffer。
#### 通道Channel
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们对数据的读取和写入要通过Channel，通道不同于流的地方就是通道是双向的，可以用于读、写和同时读写操作。底层的操作系统的通道一般都是全双工的，所以全双工的Channel比流能更好的映射底层操作系统的API。
Channel主要分两大类：
- SelectableChannel：用户网络读写
- FileChannel：用于文件操作


#### 多路选择复用器
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NIO中轮询操作是用户线程进行的，如果把这个任务交给其他线程，则用户线程就不用这么费劲的查询状态了。IO多路复用调用系统级别的select或poll模型，由系统进行监控IO状态。select轮询可以监控许多socket的IO请求，当有一个socket的数据准备好时就可以返回。<br>

![cmd-markdown-logo](http://139.224.113.197/20200120175013.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;引入了channel和多路复用器的概念，所有的连接都注册到多路复用器上，多路复用器只需要一个线程去轮询注册上来的channel，当发现channel中有IO请求的时候，才会创建或者启用一个线程。
<br>
![cmd-markdown-logo](http://139.224.113.197/20200122101735.jpg)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Selector提供选择已经就绪的任务的能力：Selector会不断轮询注册在其上的Channel，如果某个Channel上面发生读或者写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。
<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个Selector可以同时轮询多个Channel，因为JDK使用了epoll()代替传统的select实现，所以没有最大连接句柄1024/2048的限制。所以，只需要一个线程负责Selector的轮询，就可以接入成千上万的客户端。

这种模型也是有问题的：当后端的（如jdbc）操作比较耗时的时候，会发生线程占用的情况。
- 多路复用IO的特点是用户进程能同时等待多个IO请求，系统来监控IO状态，其中的任意一个进入读就绪状态，select函数就可以返回。
- select： 注册事件由数组管理, 数组是有长度的, 32位机上限1024， 64位机上限2048。轮询查找时需要遍历数组。

- poll：把select的数组采用链表实现，因此没了最大数量的限制

- epoll：基于事件回调机制，回调时直接通知进程，无须使用某种方式来查看状态。

### AIO（异步非阻塞）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AIO即Asynchronous I/O，这是Java 1.7引入的NIO 2.0中用到的。整个过程中，用户线程发起一个系统调用之后无须等待，可以处理别的事情。由操作系统等待接收内容，接收后把数据拷贝到用户进程中，最后通知用户程序已经可以使用数据了，两个阶段都是非阻塞的。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理。客户端发送请求后不需要不停的问，以及不需要等待。 等到处理结果完成后，会自动通知。AIO需要操作系统的深度支持。<br>

![cmd-markdown-logo](http://139.224.113.197/20200122101749.jpg)
<br>

- AIO的特点就是在IO执行的两个阶段都是非阻塞的。
- 核心是：**一个有效请求一个线程**。<br>

### 使用场景
- BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序直观简单易理解。
- NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持。Jetty，Mina，ZooKeeper等都是基于java nio实现。
- AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK7开始支持。

参考：<br>
[十分钟了解BIO、NIO、AIO](https://mp.weixin.qq.com/s/r-7WInsktj7hL8pYzeQ4CQ)<br>
[今天我们结合代码详细聊聊BIO，NIO和AIO](https://mp.weixin.qq.com/s/O5LUYAhp8qTw8gCk8C7_gg)<br>
[详解NIO、BIO、AIO](https://mp.weixin.qq.com/s/8LQhdaJj16NGlR-DAHLqgQ)<br>
[以Java的视角来聊聊BIO、NIO与AIO的区别？](https://mp.weixin.qq.com/s/EczCiUpae1edKR4a6sg9Cg)<br>