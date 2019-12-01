---
title: Spring Bean的完整生命周期
date: 2019-11-20 11:32:58
tags:
---

------
#### 一、Bean 的完整生命周期

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在传统的Java应用中，bean的生命周期很简单，使用Java关键字 new 进行Bean 的实例化，然后该Bean 就能够使用了。一旦bean不再被使用，则由Java自动进行垃圾回收。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;相比之下，Spring管理Bean的生命周期就复杂多了，正确理解Bean 的生命周期非常重要，因为Spring对Bean的管理可扩展性非常强，下面展示了一个Bean的构造过程
![cmd-markdown-logo](http://139.224.113.197/20191121145053.jpg)

##### Bean的生命周期
如上图所示，Bean 的生命周期还是比较复杂的，下面来对上图每一个步骤做文字描述<br>
1.&nbsp;&nbsp;Spring启动，查找并加载需要被Spring管理的bean，进行Bean的实例化<br>

2.&nbsp;&nbsp;Bean实例化后对将Bean的引入和值注入到Bean的属性中<br>

3.&nbsp;&nbsp;如果Bean实现了BeanNameAware接口的话，Spring将Bean的Id传递给setBeanName()方法<br>

4.&nbsp;&nbsp;如果Bean实现了BeanFactoryAware接口的话，Spring将调用setBeanFactory()方法，将BeanFactory容器实例传入<br>

5.&nbsp;&nbsp;如果Bean实现了ApplicationContextAware接口的话，Spring将调用Bean的setApplicationContext()方法，将bean所在应用上下文引用传入进来。<br>

6.&nbsp;&nbsp;如果Bean实现了BeanPostProcessor接口，Spring就将调用他们的postProcessBeforeInitialization()方法。<br>

7.&nbsp;&nbsp;如果Bean 实现了InitializingBean接口，Spring将调用他们的afterPropertiesSet()方法。类似的，如果bean使用init-method声明了初始化方法，该方法也会被调用<br>

8.&nbsp;&nbsp;调用<bean　init-method="">执行指定的初始化方法

9.&nbsp;&nbsp;如果Bean 实现了BeanPostProcessor接口，Spring就将调用他们的postProcessAfterInitialization()方法。<br>

10.&nbsp;&nbsp;此时，Bean已经准备就绪，可以被应用程序使用了。他们将一直驻留在应用上下文中，直到应用上下文被销毁。<br>

11.&nbsp;&nbsp;如果bean实现了DisposableBean接口，Spring将调用它的destory()接口方法，同样，如果bean使用了destory-method 声明销毁方法，该方法也会被调用<br>

12.调用<bean　destory-method="">执行自定义的销毁方法。

#### 二、Bean 的生命周期验证
验证代码如下：
```
public class BeanLifeTest implements BeanNameAware, ApplicationContextAware, InitializingBean,BeanPostProcessor, DisposableBean {

    private String name;

    public BeanLifeTest() {
        System.out.println("第一步：实例化类");
    }

    public void setName(String name) {
        System.out.println("第二步：设置属性");
        this.name = name;
    }

    @Override
    public void setBeanName(String s) {
        System.out.println("第三步：设置bean的名称也就是spring容器中的名称，也就是id值" + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("第四步：设置工厂信息ApplicationContext");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("第六步：属性设置后执行的方法");
    }

    public void init() {
        System.out.println("第七步：执行自己配置的初始化方法");
    }
    //第八步执行初始化之后执行的方法
    public void run() {
        System.out.println("第九步：执行自身的业务方法");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("第十步：执行spring的销毁方法");
    }

    public void destory() {
        System.out.println("第十一步：执行自己配置的销毁方法");
    }


}
```

```
public class BeanPostProcessTest implements BeanPostProcessor {

    //后处理bean，最重要的两步
    @Override
    public Object postProcessBeforeInitialization(Object bean, String s) throws BeansException {
        System.out.println("第五步：初始化之前执行的方法");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String s) throws BeansException {
        System.out.println("第八步：执行初始化之后的方法");
        return bean;
    }
}
```
applicationContext.xml中的配置

```
<?xml  version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <bean id="beanLife" class="com.example.demo.BeanLifeTest" init-method="init" destroy-method="destory">
        <property name="name" value="张三"/>
    </bean>

    <bean id="beanProcess" scope="prototype" class="com.example.demo.BeanPostProcessTest"></bean>
</beans>
```
测试代码如下：
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class DemoApplicationTests {

	@Test
	public void beanlifeTest() {
		ClassPathXmlApplicationContext context=new ClassPathXmlApplicationContext("applicationContext.xml");
		BeanLifeTest beanLife =(BeanLifeTest)context.getBean("beanLife");
		beanLife.run();
		BeanPostProcessTest process = (BeanPostProcessTest)context.getBean("beanProcess");
		context.close();
	}

}
```
执行结果：

```
第一步：实例化类
第二步：设置属性
第三步：设置bean的名称也就是spring容器中的名称，也就是id值张三
第四步：设置工厂信息ApplicationContext
第六步：属性设置后执行的方法
第七步：执行自己配置的初始化方法
第九步：执行自身的业务方法
第五步：初始化之前执行的方法
第八步：执行初始化之后的方法
第十步：执行spring的销毁方法
第十一步：执行自己配置的销毁方法
```


参考<br>
https://www.cnblogs.com/javazhiyin/p/10905294.html<br>
https://www.cnblogs.com/jasonboren/p/10660937.html<br>
https://blog.csdn.net/weixin_43871333/article/details/96877462




