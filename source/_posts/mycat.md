﻿---
title: MYCAT入门学习
---

# MYCAT入门学习

-----

## Mycat是什么?

&ensp;&ensp;&ensp;&ensp;从定义和分类来看，它是一个开源的分布式数据库系统，是一个实现了MySQL协议的Server，前端用户可以把它看做是一个数据库代理，用MySQL客户端工具和命令行访问，而其后端可以用MySQL原生(Native)协议与多个MySQL服务器通信，也可以用JDBC协议与大多数主流数据库服务器通信，其核心功能是分库分表，即将一个大表水平分割为N个小表，存储在后端MySQL服务器里或者其他数据库里。 Mycat发展到目前版本，已经不在是一个单纯的MySQL代理了，它的后端可以支持MySQL、SQL Server、Oracle、DB2、PostgreSQL等主流数据库，也支持MongoDB这种新型NOSQL方式的存储，未来还会支持更多类型的存储。而在最终用户看来，无论是那种存储方式，在Mycat里，都是一个传统的数据库表，支持标准的SQL语句进行数据的操作，这样一来，对前端业务系统来说，可以大幅度降低开发难度，提升开发速度，在测试阶段，可以将一表定义为任何一种Mycat支持的存储方式，比如MySQL的MyASM表、内存表、或者MongoDB、LeveIDB以及号称是世界上最快的内存数据库MemSQL上。

&ensp;&ensp;&ensp;&ensp;试想一下，用户表存放在MemSQL上，大量读频率远超过写频率的数据如订单的快照数据存放于InnoDB中，一些日志数据存放于MongoDB中，而且还能把Oracle的表跟MySQL的表做关联查询，你是否有一种不能呼吸的感觉?而未来，还能通过Mycat自动将一些计算分析后的数据灌入到Hadoop中，并能用Mycat+Storm/Spark Stream引擎做大规模数据分析，看到这里。

&ensp;&ensp;&ensp;&ensp;对于DBA来说，可以这么理解Mycat：

&ensp;&ensp;&ensp;&ensp;Mycat就是MySQL Server，而Mycat后面连接的MySQL Server，就好象是MySQL的存储引擎,如InnoDB，MyISAM等，因此，Mycat本身并不存储数据，数据是在后端的MySQL上存储的，因此数据可靠性以及事务等都是MySQL保证的，简单的说，Mycat就是MySQL最佳伴侣，它在一定程度上让MySQL拥有了能跟Oracle PK的能力。

&ensp;&ensp;&ensp;&ensp;对于软件工程师来说，可以这么理解Mycat：

&ensp;&ensp;&ensp;&ensp;Mycat就是一个近似等于MySQL的数据库服务器，你可以用连接MySQL的方式去连接Mycat(除了端口不同，默认的Mycat端口是8066而非MySQL的3306，因此需要在连接字符串上增加端口信息)，大多数情况下，可以用你熟悉的对象映射框架使用Mycat，但建议对于分片表，尽量使用基础的SQL语句，因为这样能达到最佳性能，特别是几千万甚至几百亿条记录的情况下。

&ensp;&ensp;&ensp;&ensp;对于架构师来说，可以这么理解Mycat：

&ensp;&ensp;&ensp;&ensp;Mycat是一个强大的数据库中间件，不仅仅可以用作读写分离、以及分表分库、容灾备份，而且可以用于多租户应用开发、云平台基础设施、让你的架构具备很强的适应性和灵活性，借助于即将发布的Mycat智能优化模块，系统的数据访问瓶颈和热点一目了然，根据这些统计分析数据，你可以自动或手工调整后端存储，将不同的表映射到不同存储引擎上，而整个应用的代码一行也不用改变。

&ensp;&ensp;&ensp;&ensp;当前是个大数据的时代，但究竟怎样规模的数据是和数据库系统呢?对此，国外有一个数据库领域的权威人士说了一个结论：干亿以下的数据规模仍然是数据库领域的专长，而Hadoop等这种系统，更适合的是干亿以上的规模，所以，Mycat适合1000亿条以下的单表规模，如果你的数据超过了这个规模，请投靠Mycat Plus吧!

Mycat原理

&ensp;&ensp;&ensp;&ensp;Mycat的原理并不复杂，复杂的是代码，如果代码也不复杂，那么早就成为一个传说了。 &ensp;&ensp;&ensp;&ensp;Mycat的原理中最重要的一个动词是“拦截”，它拦截了用户发送过来的SQL语句，首先对SQL语句做了一些特定的分析：如分片分析、路由分析、读写分离分析、缓存分析等，然后将此SQL发往后端的真实数据库，并将返回的结果做适当的处理，最终再返回给用户。
![file-list](https://www.2cto.com/uploadfile/Collfiles/20170904/2017090410012586.png)

&ensp;&ensp;&ensp;&ensp;上述图片里，Orders表被分为三个分片datanode(简称dn)，这三个分片是分布在两台MySQL Server上(DataHost)，即datanode=database@datahost方式，因此你可以用一台到N台服务器来分片，分片规则为(sharding rule)典型的字符串枚举分片规则，一个规则的定义是分片字段(sharding column)+分片函数(rule function)，这里的分片字段为rov而分片函数为字符串枚举方式。 当Mycat收到一个SQL时，会先解析这个SQL，查找涉及到的表，然后看此表的定义，如果有分片规则，则获取到SQL里分片字段的值，并匹配分片函数，得到该QL对应的分片列表，然后将SQL发往这些分片去执行，最后收集和处理所有分片返回的结果数据，并输出到客户端。以select * from Orders where prov=?语句为例，查到prov=wuhan，按照分片函数，wuhan返回 dn1，于是SQL就发给了MySQL1，去取DB1上的查询结果，并返回给用户。 如果上述SQL改为elect * from Orders where prov in (‘wuhan’,‘beijing’)，那么，SQL就会发给ySQL1与MySQL2去执行，然后结果集合并后输出给用户。但通常业务中我们的SQL会有Order By 以及Limit翻页语法，此时就涉及到结果集在Mycat端的二次处理，这部分的代码也比较复杂，而最复杂的则属两个表的Jion问题，为此，Mycat提出了创新性的ER分片、全局表、HBT(Human Brain Tech)人工智能的Catlet、以及结合Storm/Spark引擎等十八般武艺的解决办法，从而成为目前业界最强大的方案，这就是开源的力量!

&ensp;&ensp;&ensp;&ensp;应用场景

&ensp;&ensp;&ensp;&ensp;Mycat发展到现在，适用的场景已经很丰富，而且不断有新用户给出新的创新性的方案，以下是几个典型的应用场景： a.单纯的读写分离，此时配置最为简单，支持读写分离，主从切换 b.分表分库，对于超过1〇〇〇万的表进行分片，最大支持1 〇〇〇亿的单表分片 c.多租户应用，每个应用一个库，但应用程序只连接Mycat,从而不改造程序本身，实现多租户化 d.报表系统，借助于Mycat的分表能力，处理大规模报表的统计 e.代替Hbase,分析大数据 f.作为海量数据实时查询的一种简单有效方案，比如 1〇〇亿条频繁查询的记录需要在3秒内查询出来结果， 除了基于主键的查询，还可能存在范围查询或其他属性查询，此时Mycat可能是最简单有效的选择 —单纯的读写分离，此时配置最为简单，支持读写分离，主从切换分表分库，对于超过000万的表进行分片，最大支持1000亿的单表分片 —多租户应用，每个应用一个库，但应用程序只连接Mycat，从而不改造程序本身，实现多租户化 —报表系统，借助于Mycat的分表能力，处理大规模报表的统计替代Hbase，分析大数据，作为海量数据实时查询的一种简单有效方案，比如100亿条频繁查询的记录需要在3秒内查询出来结果，除了基于主键的查询，还可能存在范围查询或其他属性查询，此时ycat可能是最简单有效的选择

&ensp;&ensp;&ensp;&ensp;Mycat长期路线图

&ensp;&ensp;&ensp;&ensp;强化分布式数据库中间件的方面的功能，使之具备丰富的插件、强大的数据库智能优化功能、全面的系统监控能力、以及方便的数据运维工具，实现在线数据扩容、迁移等高级功能进一步挺进大数据计算领域，深度结合Spark Stream和Storm等分布式实时流引擎，能够完成快速的巨表关联、排序、分组聚合等 OLAP方向的能力，并集成一些热门常用的实时分析算法，让工程师以及DBA们更容易用Mycat实现一些高级数据分析处理功能。 不断强化Mycat开源社区的技术水平，吸引更多的IT技术专家，使得Mycat社区成为中国的Apache，并将Mycat推到Apache基金会，成为国内顶尖开源项目，最终能够让一部分志愿者成为专职的Mycat开发者，荣耀跟实力一起提升。 依托Mycat社区，聚集100个CXO级别的精英，众筹建设亲亲山庄，Mycat社区+亲亲山庄=中国最大IT O2O社区 Mycat中的概念

&ensp;&ensp;&ensp;&ensp;数据库中间件

&ensp;&ensp;&ensp;&ensp;前面讲了Mycat是一个开源的分布式数据库系统，但是由于真正的数据库需要存储引擎，而Mycat并没有存储引擎，所以并不是完全意义的分布式数据库系统。 那么Mycat是什么?Mycat是数据库中间件，就是介于数据库与应用之间，进行数据处理与交互的中间服务。由于前面讲的对数据进行分片处理之后，从原有的一个库，被切分为多个分片数据库，所有的分片数据库集群构成了整个完整的数据库存储。

![file-list](https://www.2cto.com/uploadfile/Collfiles/20170904/2017090410012589.png)

&ensp;&ensp;&ensp;&ensp;如上图所表示，数据被分到多个分片数据库后，应用如果需要读取数据，就要需要处理多个数据源的数据。如果没有数据库中间件，那么应用将直接面对分片集群，数据源切换、事务处理、数据聚合都需要应用直接处理，原本该是专注于业务的应用，将会花大量的工作来处理分片后的问题，最重要的是每个应用处理将是完全的重复造轮子。 &ensp;&ensp;&ensp;&ensp;所以有了数据库中间件，应用只需要集中与业务处理，大量的通用的数据聚合，事务，数据源切换都由中间件来处理，中间件的性能与处理能力将直接决定应用的读写性能，所以一款好的数据库中间件至关重要。

&ensp;&ensp;&ensp;&ensp;逻辑库(schema)

&ensp;&ensp;&ensp;&ensp;通常对实际应用来说，并不需要知道中间件的存在，开发人员只需要知道数据库的概念，所以数据库中间件可以被看做是一个或多个数据库集群构成的逻辑库。

&ensp;&ensp;&ensp;&ensp;在云计算时代，数据库中间件可以以多租户的形式给一个或多个应用提供服务，每个应用访问的可能是一个独立或者是共享的物理库，常见的如阿里云数据库服务器RDS。
![file-list](https://www.2cto.com/uploadfile/Collfiles/20170904/2017090410012593.png)