---
title: Spring事物管理
date: 2019-6-21 15:19:30
tags:
---

------

### 1. 事务简介
&ensp;&ensp;&ensp;&ensp;事务管理是企业级应用程序开发中必不可少的技术，用来确保数据的完整性和一致性。事务就是一系列的动作，它们被当作一个单独的工作单元。这些动作要么全部完成，要么全部不起作用。
事务的四个关键属性(ACID)，包含：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）。
- 原子性（Atomicity）：事务是一个原子操作，由一系列动作组成。事务的原子性确保动作要么全部完成，要么完全不起作用。
- 一致性（Consistency）：一旦事务完成（不管成功还是失败），系统必须确保它所建模的业务处于一致的状态，而不会是部分完成部分失败。在现实中的数据不应该被破坏。
- 隔离性（Isolation）：可能有许多事务会同时处理相同的数据，因此每个事务都应该与其他事务隔离开来，防止数据损坏。
- 持久性（Durability）：一旦事务完成，无论发生什么系统错误，它的结果都不应该受到影响，这样就能从任何系统崩溃中恢复过来。通常情况下，事务的结果被写到持久化存储器中。

### 2.  Spring事物属性
&ensp;&ensp;&ensp;&ensp;Spring在TransactionDefinition接口中定义这些属性,以供PlatfromTransactionManager使用, PlatfromTransactionManager是spring事务管理的核心接口。

```
public interface TransactionDefinition { 
    int PROPAGATION_REQUIRED = 0; //事物的传播集中（七种）
    ...
    int ISOLATION_DEFAULT = -1; //事物的隔离级别（五种）
    ...
    int getPropagationBehavior();//返回事务的传播行为。 
    int getIsolationLevel();//返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据。 
    int getTimeout();//返回事务必须在多少秒内完成。 
    boolean isReadOnly();//事务是否只读，事务管理器能够根据这个返回值进行优化，确保事务是只读的。 
}
```
#### 2.1 Spring事务的传播属性
&ensp;&ensp;&ensp;&ensp;所谓事务传播机制，也就是在事务在多个方法的调用中是如何传递的，是重新创建事务还是使用父方法的事务？父方法的回滚对子方法的事务是否有影响？这些都是可以通过事务传播机制来决定的。TransactionDefinition接口中定义了七个事务传播行为：<br>

**1）PROPAGATION_REQUIRED**<br>
如果有事务则加入事务，如果没有事务，则创建一个新的（默认值）
使用spring声明式事务，spring使用AOP来支持声明式事务，会根据事务属性，自动在方法调用之前决定是否开启一个事务，并在方法执行之后决定事务提交或回滚事务。<br>
**2）PROPAGATION_NOT_SUPPORTED**<br>
Spring不为当前方法开启事务，相当于没有事务<br>
**3）PROPAGATION_REQUIRES_NEW**<br>
不管是否存在事务，都创建一个新的事务，原来的方法挂起，新的方法执行完毕后，继续执行老的事务<br>
**4）PROPAGATION_MANDATORY**<br>
必须在一个已有的事务中执行，否则报错<br>
**5）PROPAGATION_NEVER**<br>
必须在一个没有的事务中执行，否则报错<br>
**6）PROPAGATION_SUPPORTS**<br>
如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。
如果其他bean调用这个方法时，其他bean声明了事务，则就用这个事务，如果没有声明事务，那就不用事务<br>
 **7）PROPAGATION_NESTED**<br>
如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与REQUIRED类似的操作<br>
**注意**（父方法为save，调用方法为delete）：<br>
&ensp;&ensp;&ensp;&ensp;1、当两个方法的传播机制都是REQUIRED时，如果一旦发生回滚，两个方法都会回滚<br>
&ensp;&ensp;&ensp;&ensp;2、当delete方法传播机制为REQUIRES_NEW，会开启一个新的事务，并单独提交方法，所以save方法的回滚并不影响delete方法事务提交<br>
&ensp;&ensp;&ensp;&ensp;3、当save方法为REQUIRED，delete方法为NESTED时，delete方法开启一个嵌套事务；<br>
&nbsp;&nbsp;&nbsp;&nbsp;当save方法回滚时，delete方法也会回滚；<br>&nbsp;&nbsp;&nbsp;&nbsp;反之，如果delete方法回滚，则并不影响save方法的提交。
#### 2.2 事物的隔离级别
TransactionDefinition接口中定义五个隔离级别：<br>
**ISOLATION_DEFAULT：** 这是一个PlatfromTransactionManager默认的隔离级别，使用数据库默认的事务隔离级别.另外四个与JDBC的隔离级别相对应；<br>
**ISOLATION_READ_UNCOMMITTED：** 这是事务最低的隔离级别，它充许别外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。<br>
**ISOLATION_READ_COMMITTED：**  保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。这种事务隔离级别可以避免脏读出现，但是可能会出现不可重复读和幻像读。<br>
**ISOLATION_REPEATABLE_READ：** 这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻像读。它除了保证一个事务不能读取另一个事务未提交的数据外，还保证了避免下面的情况产生(不可重复读)。<br>
**ISOLATION_SERIALIZABLE：** 这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻像读。<br>

1： Dirty reads（脏读）。也就是说，比如事务A的未提交（还依然缓存）的数据被事务B读走，如果事务A失败回滚，会导致事务B所读取的的数据是错误的。 <br>
2： non-repeatable reads（数据不可重复读）。比如事务A中两处读取数据-total-的值。在第一读的时候，total是100，然后事务B就把total的数据改成 200，事务A再读一次，结果就发现，total竟然就变成200了，造成事务A数据混乱。<br> 
3： phantom reads（幻象读数据），这个和non-repeatable reads相似，也是同一个事务中多次读不一致的问题。但是non-repeatable reads的不一致是因为他所要取的数据集被改变了（比如total的数据），但是phantom reads所要读的数据的不一致却不是他所要读的数据集改变，而是他的条件数据集改变。比如Select account.id where account.name="ppgogo*",第一次读去了6个符合条件的id，第二次读取的时候，由于事务b把一个帐号的名字由"dd"改成"ppgogo1"，结果取出来了7个数据。

#### 2.3 只读
&ensp;&ensp;&ensp;&ensp;事务的第三个特性是它是否为只读事务。如果事务只对后端的数据库进行该操作，数据库可以利用事务的只读特性来进行一些特定的优化。通过将事务设置为只读，你就可以给数据库一个机会，让它应用它认为合适的优化措施。
#### 2.4 事务超时
&ensp;&ensp;&ensp;&ensp;为了使应用程序很好地运行，事务不能运行太长的时间。因为事务可能涉及对后端数据库的锁定，所以长时间的事务会不必要的占用数据库资源。事务超时就是事务的一个定时器，在特定时间内事务如果没有执行完毕，那么就会自动回滚，而不是一直等待其结束。
#### 2.5 回滚规则
&ensp;&ensp;&ensp;&ensp;事务五边形的最后一个方面是一组规则，这些规则定义了哪些异常会导致事务回滚而哪些不会。默认情况下，事务只有遇到运行期异常时才会回滚，而在遇到检查型异常时不会回滚（这一行为与EJB的回滚行为是一致的） 
但是你可以声明事务在遇到特定的检查型异常时像遇到运行期异常那样回滚。同样，你还可以声明事务遇到特定的异常不回滚，即使这些异常是运行期异常。

#### 2.6 事务状态
&ensp;&ensp;&ensp;&ensp;上面讲到的调用PlatformTransactionManager接口的getTransaction()的方法得到的是TransactionStatus接口的一个实现，这个接口的内容如下：

```
public interface TransactionStatus{
    boolean isNewTransaction(); // 是否是新的事物
    boolean hasSavepoint(); // 是否有恢复点
    void setRollbackOnly();  // 设置为只回滚
    boolean isRollbackOnly(); // 是否为只回滚
    boolean isCompleted; // 是否已完成
}
```
### 3. Spring事物管理
&ensp;&ensp;&ensp;&ensp;Spring并不直接管理事务，而是提供了多种事务管理器，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。 Spring事务管理器的接口是org.springframework.transaction.PlatformTransactionManager，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。接口如下：

```
public interface PlatformTransactionManager {
    // 由TransactionDefinition得到TransactionStatus对象
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
    // 提交
    void commit(TransactionStatus status) throws TransactionException;
    // 回滚
    void rollback(TransactionStatus status) throws TransactionException;
}
```
&ensp;&ensp;&ensp;&ensp;从这里可知具体的事务管理机制对Spring来说是透明的，它并不关心那些，那些是对应各个平台需要关心的，所以Spring事务管理的一个优点就是为不同的事务API提供一致的编程模型，如JTA、JDBC、Hibernate、JPA。下面分别介绍各个平台框架实现事务管理的机制。
#### 3.1 JDBC事务
&ensp;&ensp;&ensp;&ensp;如果应用程序中直接使用JDBC来进行持久化，DataSourceTransactionManager会为你处理事务边界。为了使用DataSourceTransactionManager，你需要使用如下的XML将其装配到应用程序的上下文定义中：

```
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
```
实际上，DataSourceTransactionManager是通过调用java.sql.Connection来管理事务，而后者是通过DataSource获取到的。通过调用连接的commit()方法来提交事务，同样，事务失败则通过调用rollback()方法进行回滚。

#### 3.2 Hibernate事务
&ensp;&ensp;&ensp;&ensp;如果应用程序的持久化是通过Hibernate实现的，那么你需要使用HibernateTransactionManager。对于Hibernate3，需要在Spring上下文定义中添加如下的<bean>声明：

```
<bean id="transactionManager" class="org.springframework.orm.hibernate3.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory" />
</bean>
```
sessionFactory属性需要装配一个Hibernate的session工厂，HibernateTransactionManager的实现细节是它将事务管理的职责委托给org.hibernate.Transaction对象，而后者是从Hibernate Session中获取到的。当事务成功完成时，HibernateTransactionManager将会调用Transaction对象的commit()方法，反之，将会调用rollback()方法。
#### 3.3 Java持久化API事务（JPA）
&ensp;&ensp;&ensp;&ensp;Hibernate多年来一直是事实上的Java持久化标准，但是现在Java持久化API作为真正的Java持久化标准进入大家的视野。如果你计划使用JPA的话，那你需要使用Spring的JpaTransactionManager来处理事务。你需要在Spring中这样配置JpaTransactionManager：

```
<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
    <property name="sessionFactory" ref="sessionFactory" />
</bean>
```
JpaTransactionManager只需要装配一个JPA实体管理工厂（javax.persistence.EntityManagerFactory接口的任意实现）。JpaTransactionManager将与由工厂所产生的JPA EntityManager合作来构建事务。
#### 3.4 Java原生API事务
&ensp;&ensp;&ensp;&ensp;如果你没有使用以上所述的事务管理，或者是跨越了多个事务管理源（比如两个或者是多个不同的数据源），你就需要使用JtaTransactionManager：

```
<bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager">
    <property name="transactionManagerName" value="java:/TransactionManager" />
</bean>
```
JtaTransactionManager将事务管理的责任委托给javax.transaction.UserTransaction和javax.transaction.TransactionManager对象，其中事务成功完成通过UserTransaction.commit()方法提交，事务失败通过UserTransaction.rollback()方法回滚。

### 4. Spring事物管理实现方式
&ensp;&ensp;&ensp;&ensp;作为企业级应用程序框架，Spring在不同的事务管理API之上定义了一个抽象层。而应用程序开发人员不必了解底层的事务管理API，就可以使用Spring的事务管理机制。<br>
Spring既支持编程式事务管理，也支持声明式的事务管理<br>
**编程式事务管理：** 将事务管理代码嵌入到业务方法中来控制事务的提交和回滚，在编程式事务中，必须在每个业务操作中包含额外的事务管理代码，编程式事务每次实现都要单独实现，但业务量大功能复杂时，使用编程式事务无疑是痛苦的，目前这种方式已不被广泛使用。<br>
**声明式事务管理：** 大多数情况下比编程式事务管理更好用。它将事务管理代码从业务方法中分离出来，以声明的方式来实现事务管理。事务管理作为一种横切关注点，可以通过AOP方法模块化。<br>
&ensp;&ensp;&ensp;&ensp;Spring配置文件中关于事务配置总是由三个组成部分，分别是DataSource、TransactionManager和代理机制这三部分，无论哪种配置方式，一般变化的只是代理机制这部分。 DataSource、TransactionManager这两部分只是会根据数据访问方式有所变化，比如使用Hibernate进行数据访问时，DataSource实际为SessionFactory，TransactionManager的实现为HibernateTransactionManager。<br>
&ensp;&ensp;&ensp;&ensp;声明式事务实现方式主要有2种，一种为通过使用Spring的<tx:advice>定义事务通知与AOP相关配置实现，另为一种通过@Transactional实现事务管理实现，下面详细说明2种方法如何配置：<br>
方式一：

```
<tx:advice id="advice" transaction-manager="transactionManager">
	<tx:attributes>
	    <!-- 拦截save开头的方法，事务传播行为为：REQUIRED：必须要有事务, 如果没有就在上下文创建一个 -->
        <tx:method name="save*" propagation="REQUIRED" isolation="READ_COMMITTED" timeout="" read-only="false" no-rollback-for="" rollback-for=""/>
        <!-- 支持,如果有就有,没有就没有 -->
        <tx:method name="*" propagation="SUPPORTS"/>
	</tx:attributes>
</tx:advice>
<!-- 定义切入点，expression为切人点表达式，如下是指定impl包下的所有方法，具体以自身实际要求自定义  -->
<aop:config>
    <aop:pointcut expression="execution(* com.*.service.impl.*.*(..))" id="pointcut"/>
    <!--<aop:advisor>定义切入点，与通知，把tx与aop的配置关联,才是完整的声明事务配置 -->
    <aop:advisor advice-ref="advice" pointcut-ref="pointcut"/>
```
方式二：

```
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
<!-- 开启事务注解驱动 在业务逻辑层上使用@Transactional 注解为业务逻辑层管理事务-->
<tx:annotation-driven  transaction-manager="transactionManager"/>
```
