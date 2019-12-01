---
title: Spring常见知识点
date: 2019-11-26 17:30:53
tags:
---

------
#### 一、什么是Spring的依赖注入，有哪些方法进行依赖注入
依赖注入是IOC的一个方法，就是调用者不用创建对象，只需要描述对象如何创建，描述哪些组件需要哪些服务，之后IOC容器把他们组装起来。<br>
**构造器依赖注入**：通过容器触发一个类的构造器来实现，该类有一系列参数，每一个参数代表一个对其他类的依赖。<br>
**setter方法依赖注入**：容器在实例化bean后，调用bean的setter方法，即实现基于setter的依赖注入。

#### 二、 在 Spring中如何注入一个java集合？
Spring提供以下几种集合的配置元素：<br>
<list>类型用于注入一列值，允许有相同的值<br>
<set> 类型用于注入一组值，不允许有相同的值<br>
<map> 类型用于注入一组键值对，键和值都可以为任意类型<br>
<props>类型用于注入一组键值对，键和值都只能为String类型<br>

```
<property name="excludeUrls">
   <list>
      <value>/jsp</value>
      <value>/plat</value>
   </list>
</property>
```
#### 三、Spring 初始化bean的时机
1、使用BeanFactory作为Spring Bean的工厂类时，所有的Bean都会在第一次使用时（getBean）初始化<br>
2、使用ApplicationContext作为SpringBean的工厂类时<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a、bean的scope是singleton时，且lazy-init为false时（默认为false）,ApplicationContext启动时就会初始化，并将实例化的bean放在一个map结构的缓存中，下次再使用这个bean时，直接从缓存中取<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b、bean的scope是singleton时，且lazy-init为true时，则该bean的实例化是在该bean第一次使用时（getBean）进行实例化<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;c、bean的scope是prototype时，则该bean的实例化是在该bean第一次使用时（getBean）进行实例化

#### 四、Spring如何解决循环依赖
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、构造器参数循环依赖-无解。Spring把正在创建的Bean标示符放在“当前创建bean池”中，A依赖B，B依赖C，C依赖A，创建A，把A放在池中去创建B..,创建C时发现A已经在池中，就会报错。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、field属性注入依赖，单例模式-有解。Spring先用构造器实例化对象，此时Spring会将实例化结束的对象放在Map中，并提供了获取这个未设置属性的实例化对象引用的方法。Spring实例化A、B、C后紧接着去设置对象属性，A依赖B，从Map中取出B的对象，依此解决循环依赖。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、field属性注入依赖，原型模式-无解。Prototype作用的Bean，Spring容器无法完成依赖注入，因为prototype作用对象，Spring不会进行缓存，无法提前暴露一个创建中的Bean。<br>
[参考：https://www.cnblogs.com/tiger-fu/p/8961361.html](https://www.cnblogs.com/tiger-fu/p/8961361.html)

#### 五、SpringMVC核心处理流程

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1、DispatcherServlet前端控制器接收发过来的请求，交给HandlerMapping处理器映射器<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2、HandlerMapping处理器映射器，根据请求路径找到相应的HandlerAdapter处理器适配器（处理器适配器就是那些拦截器或Controller）<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3、HandlerAdapter处理器适配器，处理一些功能请求，返回一个ModelAndView对象（包括模型数据、逻辑视图名）<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4、ViewResolver视图解析器，先根据ModelAndView中设置的View解析具体视图<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5、然后再将Model模型中的数据渲染到View上<br>
这些过程都是以DispatcherServlet为中轴线进行的

#### 六、SpringBean的生命周期
Spring 容器中的bean的完整生命周期一共分为十一步完成。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.bean对象的实例化<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.封装属性，也就是设置properties中的属性值<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.如果bean实现了BeanNameAware，则执行setBeanName方法,也就是bean中的id值<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4.如果实现BeanFactoryAware或者ApplicationContextAware ，需要设置setBeanFactory或者上下文对象setApplicationContext<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5.如果存在类实现BeanPostProcessor后处理bean，执行postProcessBeforeInitialization，可以在初始化之前执行一些方法<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6.如果bean实现了InitializingBean，则执行afterPropertiesSet，执行属性设置之后的操作<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;7.调用<bean　init-method="">执行指定的初始化方法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8.如果存在类实现BeanPostProcessor则执行postProcessAfterInitialization，执行初始化之后的操作<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;9.执行自身的业务方法<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;10.如果bean实现了DisposableBean，则执行spring的的销毁方法<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;11.调用<bean　destory-method="">执行自定义的销毁方法。<br>
第五步和第八步可以结合aop，在初始化执行之前或者执行之后执行一些操作。<br>

#### 七、说一下Spring中Bean的作用域
**singleton:** Spring默认的作用域，springIOC容器中只会存在一个共享的Bean实例<br>
**prototype:**
每次从spring容器获取bean时都会创建并返回一个新的实例，每个bean都有自己的属性和状态<br>
**reqeust:**
在一次http请求中，容器返回该bean的同一个实例，该bean仅在http request中有效，不同的http request会产生新的bean<br>
**session:**
在一次http session请求中，容器返回该bean的同一个实例，该bean仅在session中有效，不同的session会产生新的bean<br>
**global session:**
在一个全局的http session中，容器会返回同一个实例，仅在使用portlet context时有效

#### 八、Spring框架中都用到了哪些设计模式
**单例模式：**
spring配置文件中定义的bean默认为单例模式<br>
**工厂模式：**
beanFactory用来创建对象的实例<br>
**代理模式：**
在AOP和remoting中用的比较多<br>
**依赖注入模式：**
贯穿BeanFactory/ApplicationContext接口的核心概念<br>
**模板方法模式：**
用来解决代码重复问题<br>

#### 九、BeanFactory 和ApplicationContext的区别
1、BeanFactory和ApplicationContext都是接口，ApplicationContext继承了BeanFactory<br>
2、BeanFactory是Spring中最底层的接口，提供了最简单的容器功能，只提供了实例化对象和拿对象的功能。ApplicationContext是Spring更高级的容器，提供了更多的有用功能。<br>
3、加载方式的区别：BeanFactory采用的延迟加载的方式注入bean，ApplicationContext对于单例模式的bean在IOC容器启动的时候就一次性创建所有的bean,好处是可以马上发现Spring配置文件中的错误，坏处是造成浪费。<br>
4、ApplicationContext提供的额外的功能：国际化的功能、消息发送、响应机制、统一加载资源的功能、强大的事件机制、对Web应用的支持等等。<br>

#### 十、ApplicationContext的通常实现有哪些
FileSystemXmlApplicationContext：Spring容器通过xml文件的全路径，从一个xml文件中加载bean的定义，如

```
ApplicationContext context = new FileSystemXmlApplicationContext("/src/main/resources/bean.xml");
```

ClassPathXmlApplicationContext：Spring容器从classpath下的xml文件中加载beans的定义

```
ClassPathXmlApplicationContext context=new ClassPathXmlApplicationContext("applicationContext.xml");
```

WebXmlApplicationContext：Spring容器加载一个xml文件，该文件定义了Web应用的所有beans。

#### 十一、Spring的优点
1、降低了组件之间的耦合性 ，实现了软件各层之间的解耦 <br>
2、可以使用容易提供的众多服务，如事务管理，消息服务等 <br>
3、容器提供单例模式支持 <br>
4、容器提供了AOP技术，利用它很容易实现如权限拦截，运行期监控等功能 <br>
5、容器提供了众多的辅助类，能加快应用的开发 <br>
6、spring对于主流的应用框架提供了集成支持，如hibernate，JPA，Struts等 <br>
7、spring属于低侵入式设计，代码的污染极低 <br>
8、独立于各种应用服务器 <br>
9、spring的DI机制降低了业务对象替换的复杂性 <br>
10、Spring的高度开放性，并不强制应用完全依赖于Spring，开发者可以自由选择spring的部分或全部 <br>

#### 十二、Spring框架的事物管理有哪些优点
1、它为不同的事物，如JTA、JDBC、Hibernate、JPA和JDO通过了一个不变的编程模式<br>
2、它为编程式事物提供了一套简单的API而不是一套复杂的API<br>
3、支持声明式事物管理<br>
4、它和Spring各数据访问抽象层很好的集成

#### 十三、列举一下你知道的实现spring事务的几种方式
1、编程式事务管理：需要手动编写代码，在实际开发中很少使用<br>
2、基于TransactionProxyFactoryBean的声明式事务管理，需要为每个进行事务管理的类做相应配置<br>
3、基于AspectJ的XML的声明式事务管理，不需要改动类，在XML文件中配置好即可<br>
4、基于注解的声明式事务管理，配置简单，需要在业务层类中添加注解

#### 十四、谈谈Spring事务的隔离级别和传播行为
**隔离级别**：<br>
    - DEFAULT使用数据库默认的隔离级别<br>
    - READ_UNCOMMITTED会出现脏读，不可重复读和幻影读问题<br>
    - READ_COMMITTED会出现重复读和幻影读<br>
    - REPEATABLE_READ会出现幻影读<br>
    - SERIALIZABLE最安全，但是代价最大，性能影响极其严重<br>
**传播行为**：<br>
    - REQUIRED存在事务就融入该事务，不存在就创建事务<br>
    - SUPPORTS存在事务就融入事务，不存在则不创建事务<br>
    - MANDATORY存在事务则融入该事务，不存在，抛异常<br>
    - REQUIRES_NEW总是创建新事务<br>
    - NOT_SUPPORTED存在事务则挂起，一直执行非事务操作<br>
    - NEVER总是执行非事务，如果当前存在事务则抛异常<br>
    - NESTED嵌入式事务<br>


#### 十四、谈谈你对Spring的理解
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.Spring是实现了工厂模式的工厂类（在这里有必要解释清楚什么是工厂模式），这个类名为BeanFactory（实际上是一个接口），在程序中通常BeanFactory的子类ApplicationContext。Spring相当于一个大的工厂类，在其配置文件中通过<bean>元素配置用于创建实例对象的类名和实例对象的属性。<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2. Spring提供了对IOC良好支持，IOC是一种编程思想，是一种架构艺术，利用这种思想可以很好地实现模块之间的解耦，IOC也称为DI（Depency Injection）。<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3. Spring提供了对AOP技术的良好封装， AOP称为面向切面编程，就是系统中有很多各不相干的类的方法，在这些众多方法中要加入某种系统功能的代码，例如，加入日志，加入权限判断，加入异常处理，这种应用称为AOP。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;实现AOP功能采用的是代理技术，客户端程序不再调用目标，而调用代理类，代理类与目标类对外具有相同的方法声明，有两种方式可以实现相同的方法声明，一是实现相同的接口，二是作为目标的子类。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在JDK中采用Proxy类产生动态代理的方式为某个接口生成实现类，如果要为某个类生成子类，则可以用CGLIB。在生成的代理类的方法中加入系统功能和调用目标类的相应方法，系统功能的代理以Advice对象进行提供，显然要创建出代理对象，至少需要目标类和Advice类。spring提供了这种支持，只需要在spring配置文件中配置这两个元素即可实现代理和aop功能。

