---
title: Redis使用总结
date: 2019-11-28 19:22:18
tags: Redis
---

------
#### 一、redis简介
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis是一个开源的内存中的数据结构存储系统，它可以用作：数据库、缓存和消息中间件。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;它支持多种类型的数据结构，如字符串（String），散列（Hash），列表（List），集合（Set），有序集合（Sorted Set或者是ZSet）与范围查询，Bitmaps，Hyperloglogs 和地理空间（Geospatial）索引半径查询。其中常见的数据结构类型有：String、List、Set、Hash、ZSet这5种。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 内置了复制（Replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（Transactions） 和不同级别的磁盘持久化（Persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（High Availability）。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 内置了复制（Replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（Transactions） 和不同级别的磁盘持久化（Persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（High Availability）。

#### 二、redis为什么会这么快
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis采用的是基于内存的采用的是单进程单线程模型的 KV 数据库，由C语言编写，官方提供的数据是可以达到100000+的QPS（每秒内查询次数）。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、完全基于内存，绝大部分请求是纯粹的内存操作，非常快速。数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的；<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4、使用多路I/O复用模型，非阻塞IO；<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;多路I/O复用模型是利用 select、poll、epoll 可以同时监察多个流的 I/O 事件的能力，在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有 I/O 事件时，就从阻塞态中唤醒，于是程序就会轮询一遍所有的流（epoll 是只轮询那些真正发出了事件的流），并且只依次顺序的处理就绪的流，这种做法就避免了大量的无用操作。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5、使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；

#### 三、redis的过期策略以及内存淘汰机制

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**分析：** 这个问题其实相当重要，到底redis有没用到家，这个问题就可以看出来。比如你redis只能存5G数据，可是你写了10G，那会删5G的数据。怎么删的，这个问题思考过么？还有，你的数据已经设置了过期时间，但是时间到了，内存占用率还是比较高，有思考过原因么?<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**回答:**<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;redis采用的是定期删除+惰性删除策略。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为什么不用定时删除策略?<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;定时删除,用一个定时器来负责监视key,过期则自动删除。虽然内存及时释放，但是十分消耗CPU资源。在大并发请求下，CPU要将时间应用在处理请求，而不是删除key,因此没有采用这一策略。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;定期删除+惰性删除是如何工作的呢?<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;定期删除，redis默认每个100ms检查，是否有过期的key,有过期key则删除。需要说明的是，redis不是每个100ms将所有的key检查一次，而是随机抽取进行检查(如果每隔100ms,全部key进行检查，redis岂不是卡死)。因此，如果只采用定期删除策略，会导致很多key到时间没有删除。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;于是，惰性删除派上用场。也就是说在你获取某个key的时候，redis会检查一下，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;采用定期删除+惰性删除就没其他问题了么?<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不是的，如果定期删除没删除key。然后你也没即时去请求key，也就是说惰性删除也没生效。这样，redis的内存会越来越高。那么就应该采用内存淘汰机制。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在redis.conf中有一行配置
```
#maxmemory-policy volatile-lru
```

该配置就是配内存淘汰策略的<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**noeviction：** 当内存不足以容纳新写入数据时，新写入操作会报错。应该没人用吧。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**allkeys-lru：** 当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。推荐使用，目前项目在用这种。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**allkeys-random：** 当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。应该也没人用吧，你不删最少使用Key,去随机删。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**volatile-lru：** 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。这种情况一般是把redis既当缓存，又做持久化存储的时候才用。不推荐<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**volatile-random：** 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。依然不推荐<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**volatile-ttl：** 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。不推荐<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ps：如果没有设置 expire 的key, 不满足先决条件(prerequisites); 那么 volatile-lru, volatile-random 和 volatile-ttl 策略的行为, 和 noeviction(不删除) 基本上一致。

#### 四、持久化
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;持久化是Redis高可用中比较重要的一个环节，因为Redis数据在内存的特性，持久化必须有，Redis持久化主要有两种方式：<br>
##### 1、RDB（redis database）<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。<br>

**RDB持久化配置：**<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis会将数据集的快照dump到dump.rdb文件中。此外，我们也可以通过配置文件来修改Redis服务器dump快照的频率，在打开6379.conf文件之后，我们搜索save，可以看到下面的配置信息：<br>

```
save 900 1      #在900秒(15分钟)之后，如果至少有1个key发生变化，则dump内存快照。
save 300 10     #在300秒(5分钟)之后，如果至少有10个key发生变化，则dump内存快照。
save 60 10000   #在60秒(1分钟)之后，如果至少有10000个key发生变化，则dump内存快照。
```


##### 2、AOF（append only file）<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AOF持久化将每一个写、删除操作，以日志 append-only 的模式写入一个日志文件中，因为这个模式是只追加的方式，所以没有任何磁盘寻址的开销，所以很快，有点像Mysql中的binlog。AOF文本的方式记录，可以打开文件看到详细的操作记录。<br>

**AOF持久化配置:**<br>
在Redis的配置文件中存在三种同步方式，它们分别是：

```
appendfsync always    #每次有数据修改发生时都会写入AOF文件。
appendfsync everysec  #每秒钟同步一次，该策略为AOF的缺省策略。
appendfsync no        #从不同步。高效但是数据不会被持久化。
```

两种方式都可以把Redis内存中的数据持久化到磁盘上，然后再将这些数据备份到别的地方去，RDB更适合做冷备，AOF更适合做热备。两种机制全部开启的时候，Redis在重启的时候会默认使用AOF去重新构建数据，因为AOF的数据是比RDB更完整的。

##### 3、RDB和AOF优缺点<br>

**RDB存在哪些优势呢？**<br>

1).  对于灾难恢复而言，RDB是非常不错的选择。因为我们可以非常轻松的将一个单独的文件压缩后再转移到其它存储介质上；<br>

2).  性能最大化，对于Redis的服务进程而言，在开始持久化时，它唯一需要做的只是fork出子进程，之后再由子进程完成这些持久化的工作，这样就可以极大的避免服务进程执行IO操作了；<br>

3).  相比于AOF机制，如果数据集很大，RDB的启动效率会更高。

**RDB又存在哪些劣势呢？**<br>

1).  如果你想保证数据的高可用性，即最大限度的避免数据丢失，那么RDB将不是一个很好的选择。因为系统一旦在定时持久化之前出现宕机现象，此前没有来得及写入磁盘的数据都将丢失；<br>

2).  由于RDB是通过fork子进程来协助完成数据持久化工作的，因此，如果当数据集较大时，可能会导致整个服务器停止服务几百毫秒，甚至是1秒钟。<br>

**AOF的优势有哪些呢？**<br>

1).  该机制可以带来更高的数据安全性，即数据持久性。Redis中提供了3中同步策略，即每秒同步、每修改同步和不同步。事实上，每秒同步也是异步完成的，其效率也是非常高的，所差的是一旦系统出现宕机现象，那么这一秒钟之内修改的数据将会丢失。而每修改同步，我们可以将其视为同步持久化，即每次发生的数据变化都会被立即记录到磁盘中。可以预见，这种方式在效率上是最低的。至于无同步，无需多言，我想大家都能正确的理解它。<br>

2). 由于该机制对日志文件的写入操作采用的是append模式，因此在写入过程中即使出现宕机现象，也不会破坏日志文件中已经存在的内容。然而如果我们本次操 作只是写入了一半数据就出现了系统崩溃问题，不用担心，在Redis下一次启动之前，我们可以通过redis-check-aof工具来帮助我们解决数据 一致性的问题。

3). 如果日志过大，Redis可以自动启用rewrite机制。即Redis以append模式不断的将修改数据写入到老的磁盘文件中，同时Redis还会创 建一个新的文件用于记录此期间有哪些修改命令被执行。因此在进行rewrite切换时可以更好的保证数据安全性。

4).  AOF包含一个格式清晰、易于理解的日志文件用于记录所有的修改操作。事实上，我们也可以通过该文件完成数据的重建。

**AOF的劣势有哪些呢？**<br>

1). 对于相同数量的数据集而言，AOF文件通常要大于RDB文件。RDB 在恢复大数据集时的速度比AOF的恢复速度要快。

2). 根据同步策略的不同，AOF在运行效率上往往会慢于RDB。总之，每秒同步策略的效率是比较高的，同步禁用策略的效率和RDB一样高效。

二者选择的标准，就是看系统是愿意牺牲一些性能，换取更高的缓存一致性（aof），还是愿意写操作频繁的时候，不启用备份来换取更高的性能，待手动运行save的时候，再做备份（rdb）。rdb这个就更有些 eventually consistent的意思了。这种策略可以同时使用，redis宕机时可以第一时间用RDB恢复，然后用AOF数据补全。

#### 五、Redis集群
##### 1、哨兵模式（Redis Sentinal）<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis Sentinal 着眼于高可用，在master宕机时会自动将slave提升为master，继续提供服务。<br>
**集群监控：** 负责监控 Redis master 和 slave 进程是否正常工作。<br>
**消息通知**：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。<br>
**故障转移：** 如果 master node 挂掉了，会自动转移到 slave node 上。<br>
**配置中心：** 如果故障转移发生了，通知 client 客户端新的 master 地址。


##### 2、集群模式（Redis Cluster）<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis Cluster 着眼于扩展性，在单个redis内存不足时，使用Cluster进行分片存储。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;类似Mysql的主从同步，Redis cluster 支撑 N 个 Redis master node，每个master node都可以挂载多个 slave node。Redis集群 就可以横向扩容了。如果你要支撑更大数据量的缓存，那就横向扩容更多的 master 节点，每个 master 节点就能存放更多的数据。

##### 3、Redis主从数据同步<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一台slave机器启动的时候，它会发送一个psync命令给master ，如果是这个slave第一次连接到master，他会触发一个全量复制。master就会启动一个线程，生成RDB快照，还会把后续的写请求都缓存在内存中，RDB文件生成后，master会将这个RDB发送给slave的，slave拿到之后做的第一件事情就是写进本地的磁盘，然后加载进内存，然后master会把内存里面缓存的那些新命令都发给slave。后续的增量数据通过AOF日志同步即可，有点类似数据库的binlog。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;传输过程中网络断开时，redis会自动重连，并在连接之后把缺少的数据补上。


#### 六、使用redis有什么缺点
基本上使用redis都会碰到一些问题，常见的主要有五个问题：<br>

##### 1、缓存和数据库双写一致性问题<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**分析:** 一致性问题是分布式常见问题，还可以再分为最终一致性和强一致性。数据库和缓存双写，就必然会存在不一致的问题。答这个问题，先明白一个前提。就是如果对数据有强一致性要求，不能放缓存。我们所做的一切，只能保证最终一致性。另外，我们所做的方案其实从根本上来说，只能说降低不一致发生的概率，无法完全避免。因此，有强一致性要求的数据，不能放缓存。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**回答:** 《分布式之数据库和缓存双写一致性方案解析》给出了详细的分析，在这里简单的说一说。首先，采取正确更新策略，先更新数据库，再删缓存。其次，因为可能存在删除缓存失败的问题，提供一个补偿措施即可，例如利用消息队列。<br>

##### 2、缓存雪崩问题<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;即缓存同一时间大面积的失效，这个时候又来了一波请求，结果请求都怼到数据库上，从而导致数据库连接异常甚至宕机。<br>

**解决方案:**<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、给缓存的失效时间，加上一个随机值，避免集体失效。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、使用互斥锁，但是该方案吞吐量明显下降了。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、设置热点数据永远不过期，有更新操作就更新缓存<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4、双缓存。我们有两个缓存，缓存A和缓存B。缓存A的失效时间为20分钟，缓存B不设失效时间。自己做缓存预热操作。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;然后细分以下几个小点：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;I 从缓存A读，有则直接返回<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;II A没有数据，直接从B读数据，直接返回，并且异步启动一个更新线程。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;III 更新线程同时更新缓存A和缓存B。<br>

##### 3、缓存击穿问题<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;缓存击穿跟缓存雪崩有点像，但是又有一点不一样，缓存雪崩是因为大面积的缓存失效，打崩了DB，而缓存击穿穿是指一个Key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个Key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个完好无损的桶上凿开了一个洞。

**解决方案:**<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、设置热点数据缓存时间足够长，数据库更新时更新缓存<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、利用互斥锁，缓存失效的时候，先去获得锁，得到锁了，再去请求数据库。没得到锁，则休眠一段时间重试<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、在Nginx代理层增加限流，IP或者接口请求频率超过阀值时直接返回<br>

##### 4、缓存穿透问题<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;即攻击者故意去请求缓存中不存在，数据库中也没有的数据，导致所有的请求都怼到数据库上，从而数据库连接异常。<br>

**解决方案:**<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、网关层增加安全验证比如用户鉴权、消息验签，接口层增加参数验证，不合法的请求直接返回<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、缓存没命中，数据库中也找不到值时，可以将缓存key对应的value写成null、稍后重试这样，缓存时间可以设置短点，比如5s。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、提供一个能迅速判断请求是否有效的拦截机制，比如，利用布隆过滤器，内部维护一系列合法有效的key。迅速判断出，请求所携带的Key是否合法有效。如果不合法，则直接返回。<br>

##### 5、缓存的并发竞争问题<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**分析:** 这个问题大致就是，同时有多个子系统去set一个key。这里不推荐使用redis的事务机制。因为我们的生产环境，基本都是redis集群环境，做了数据分片操作。你一个事务中有涉及到多个key操作的时候，这多个key不一定都存储在同一个redis-server上。因此，redis的事务机制，十分鸡肋。<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**回答:** 如下所示：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、如果对这个key操作，不要求顺序。这种情况下，准备一个分布式锁，大家去抢锁，抢到锁就做set操作即可，比较简单。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、如果对这个key操作，要求顺序<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;假设有一个key1,系统A需要将key1设置为valueA,系统B需要将key1设置为valueB,系统C需要将key1设置为valueC.
期望按照key1的value值按照 valueA-->valueB-->valueC的顺序变化。这种时候我们在数据写入数据库的时候，需要保存一个时间戳。<br>
假设时间戳如下<br>
```
系统A key 1 {valueA  3:00}
系统B key 1 {valueB  3:05}
系统C key 1 {valueC  3:10}
```
那么，假设这会系统B先抢到锁，将key1设置为{valueB 3:05}。接下来系统A抢到锁，发现自己的valueA的时间戳早于缓存中的时间戳，那就不做set操作了。以此类推。
其他方法，比如利用队列，将set方法变成串行访问也可以。总之，灵活变通。

##### 缓存其他问题：<br>

什么是缓存的二八定律？<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;80%的业务访问集中在20%的数据上	

什么是热数据和冷数据？<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;冷数据是较长时间之前的状态数据，即用户画像数据，常见的有银行凭证、税务凭证、医疗档案、影视资料等<br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;热数据指即时的位置状态、交易和浏览行为。如即时的地理位置，某一特定时间活跃的手机应用等，能够表征“正在什么位置干什么事情”。另外一些实时的记录信息，如用户刚刚打开某个软件或者网站进行了一些操作，热数据可以通过第三方平台去积累，开发者也可以根据用户使用行为积累。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;温数据是非即时的状态和行为数据。简单理解可以这样，把热数据和冷数据混在一起就成了温数据。比如用户近期对某一类型的话题特别感兴趣（热数据），与以往的行为（冷数据）形成鲜明对比，这说明该用户正处于新用户的成长期（温数据），运营人员就可以考虑用相应的策略去拉动活跃度并促进转化。

#### 七、扩展
1、单进程多线程模型：MySQL、Memcached、Oracle（Windows版本）；<br>

2、多进程模型：Oracle（Linux版本）；<br>

3、Nginx有两类进程，一类称为Master进程(相当于管理进程)，另一类称为Worker进程（实际工作进程）。启动方式有两种：<br>

（1）单进程启动：此时系统中仅有一个进程，该进程既充当Master进程的角色，也充当Worker进程的角色。<br>
（2）多进程启动：此时系统有且仅有一个Master进程，至少有一个Worker进程工作。<br>
（3）Master进程主要进行一些全局性的初始化工作和管理Worker的工作；事件处理是在Worker中进行的。

参考：<br>
[《吊打面试官》系列- Redis基础](https://juejin.im/post/5db66ed9e51d452a2f15d833)<br>
[redis持久化的几种方式](https://www.cnblogs.com/chenliangcl/p/7240350.html)<br>
[深入剖析Redis - Redis集群模式搭建与原理详解](https://www.jianshu.com/p/84dbb25cc8dc)
