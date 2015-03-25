---
layout: post
title: Spring tips
category: java技术
published: true
---
 
# 1. Spring中bean的 id 和 name

同名bean：多个bean 有相同的 name 或者 id，称之为同名bean 

<bean> 的id 和 name的区别 

id和name都是spring 容器中中bean 的唯一标识符。 

id: 一个bean的唯一标识  ， 命名格式必须符合XML ID属性的命名规范 

name: 可以用特殊字符，并且一个bean可以用多个名称：name=“bean1,bean2,bean3” ,用逗号或者分号或者空格隔开。如果没有id，则name的第一个名称默认是id 

spring 容器如何处理同名bean？ 

* 同一个spring配置文件中，bean的 id、name是不能够重复的，否则spring容器启动时会报错。 

* 如果一个spring容器从多个配置文件中加载配置信息，则多个配置文件中是允许有同名bean的，并且后面加载的配置文件的中的bean定义会覆盖前面加载的同名bean。 

spring 容器如何处理没有指定id、name属性的bean？ 

* 如果 一个 <bean> 标签未指定 id、name 属性，则 spring容器会给其一个默认的id，值为其类全名。 

* 如果有多个<bean> 标签未指定 id、name 属性，则spring容器会按照其出现的次序，分别给其指定 id 值为 "类全名#0", "类全名#1", "类全名#2" 

配置文件： 

```java
<bean class="com.xxx.UserInfo">  
    <property name="accountName" value="no-id-no-name0"></property>  
</bean>  
  
<bean class="com.xxx.UserInfo">  
    <property name="accountName" value="no-id-no-name1"></property>  
</bean>  
  
<bean class="com.xxx.UserInfo">  
    <property name="accountName" value="no-id-no-name2"></property>  
</bean>  
```
   
获取bean的方式： 

```java
UserInfo u4 = (UserInfo)ctx.getBean("com.xxx.UserInfo#0");  
UserInfo u5 = (UserInfo)ctx.getBean("com.xxx.UserInfo#1");  
UserInfo u6 = (UserInfo)ctx.getBean("com.xxx.UserInfo#2");
```

# 2. spring bean id重复覆盖的问题

问题：
   当我们的web应用做成一个大项目之后，里面有很多的bean配置，如果两个bean的配置id是一样的而且实现类也是一样的，例如有下面两份xml的配置文档:

beancontext1.xml

```java
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "/spring-beans.dtd">  
<beans>  
    <bean id="testbean" class="com.xxx.Bean">  
        <property name="name" value="beancontext1" />  
    </bean>  
</beans>  
```java

beancontext2.xml

```java
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "/spring-beans.dtd">  
<beans>  
    <bean id="testbean" class="com.xxx.Bean">  
        <property name="name" value="beancontext2" />  
    </bean>  
</beans>  
```
  
当spring容器初始化时候同时加载这两份配置文件到当前的上下文的时候，代码如下：

```java
public static void main(String[] args) {  

    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(  
            new String[] {  
                    "com/config/spring/beancontext1.xml",  
                    "com/config/spring/beancontext2.xml" });  

    //context.setAllowBeanDefinitionOverriding(false);  
    //context.refresh();  
    Bean bean = (Bean) context.getBean("testbean");  
    System.out.println(bean.getName());  
}  
```
 执行这个程序你会看见控制台上打印的结果是：
beancontext2
显然，beancontext2.xml的bean的配置覆盖了 beancontext1.xml中bean的配置，而且在spring初始化上下文的过程中这个过程是静悄悄的执行的，连一点警告都没有。这样如果你的项目中定义了两个id同名的bean，并且，他们的实现方式又是不一样的，这样在后期在项目中执行的逻辑看起来就会非常诡异，而且，如果有大量配置spring配置文件的话，排查问题就会非常麻烦。

spring在处理有重名的bean的定义的时候使用的覆盖（override）的方式。我们来看看它是如何覆盖的
在org.springframework.beans.factory.support.DefaultListableBeanFactory 这个类中有这样一段代码：

```java
synchronized (this.beanDefinitionMap) {
  Object oldBeanDefinition = this.beanDefinitionMap.get(beanName);
  if (oldBeanDefinition != null) {
    if (!this.allowBeanDefinitionOverriding) {
      throw new BeanDefinitionStoreException(beanDefinition
          .getResourceDescription(), beanName,
          "Cannot register bean definition ["
              + beanDefinition + "] for bean '"
              + beanName + "': There is already ["
              + oldBeanDefinition + "] bound.");
    } else {
      if (this.logger.isInfoEnabled()) {
        this.logger
            .info("Overriding bean definition for bean '"
                + beanName + "': replacing ["
                + oldBeanDefinition + "] with ["
                + beanDefinition + "]");
      }
    }
  } else {
    this.beanDefinitionNames.add(beanName);
    this.frozenBeanDefinitionNames = null;
  }
  this.beanDefinitionMap.put(beanName, beanDefinition);
  resetBeanDefinition(beanName);
}
```

spring ioc容器在加载bean的过程中会去判断beanName 是否有重复，如果发现重复的话在根据allowBeanDefinitionOverriding 这个成员变量，如果是false的话则抛出BeanDefinitionStoreException 这个异常，如果为true的话就会覆盖这个bean的定义（默认为true）。
所以，解决这个问题的办法就比较简单了，只要将这个allowBeanDefinitionOverriding值在spring初始化的时候设置为false就行了。

在web工程中加载spring容器会通过:

```java
<listener>  
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
</listener>  
```

这个listener来完成的，在这个listener中会构造 org.springframework.web.context.ContextLoader 这个构造器来加载bean

所以，只要扩展 ContextLoader 和ContextLoaderListener这两个类就行了，代码如下：

```java  
public class XXXContextLoader extends ContextLoader {  
  
    @Override  
    protected void customizeContext(ServletContext servletContext,  
            ConfigurableWebApplicationContext applicationContext) {  
        XmlWebApplicationContext context = (XmlWebApplicationContext) applicationContext;  
        context.setAllowBeanDefinitionOverriding(false);  
    }  
  
}  
```

```java
public class XXXContextLoaderListener extends ContextLoaderListener {  
  
    @Override  
    protected ContextLoader createContextLoader() {  
        return new XXXContextLoader();  
    }  
  
}  
```

最后修改wen-inf 下web.xml 文件，修改listener的配置，如下：

```java
<listener>  
        <listener-class>com.xxx.springcontext.XXXContextLoaderListener</listener-class>  
</listener>  
```

设置完这些就ok了，这样你项目中如果在两份被加载的xml文件中如果再出现名字相同的bean的话，spring在加载过程中就会无情的抛出异常。




