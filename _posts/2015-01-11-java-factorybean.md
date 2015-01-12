---
layout: post
title: FactoryBean
category: java技术
published: true
---

## 1.概述 

Spring中有两种类型的Bean，一种是普通Bean，另一种是工厂Bean，即FactoryBean，这两种Bean都被容器管理，但工厂Bean跟普通Bean不同，其返回的对象不是指定类的一个实例，其返回的是该FactoryBean的getObject方法所返回的对象。在Spring框架内部，有很多地方有FactoryBean的实现类，它们在很多应用如(Spring的AOP、ORM、事务管理)及与其它第三框架(ehCache)集成时都有体现，下面简单分析FactoryBean的用法。 

## 2.实例 

以下SimpleFactoryBean类实现了FactoryBean接口中的三个方法。 并将该类配置在XML中。 

```java
public class MyFactoryBean implements FactoryBean {
    private boolean flag;

    public Object getObject() throws Exception {
        if (flag) {
            return new Date();
        }
        return new String();
    }

    @SuppressWarnings("unchecked")
    public Class getObjectType() {
        return flag ? Date.class : String.class;
    }

    public boolean isSingleton() {
        return false;
    }

    public void setFlag(boolean flag) {
        this.flag = flag;
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "/spring-beans.dtd">
<beans>
    <bean id="factoryBeanOne" class="com.demo.factorybean.MyFactoryBean" >
        <property name="flag">
            <value>true</value>
        </property>
    </bean>

    <bean id="factoryBeanTwo" class="com.demo.factorybean.MyFactoryBean" >
    <property name="flag">
        <value>false</value>
    </property>
    </bean>
</beans>
```


```java
public class Test {
    public static void main(String[] args) {
        Resource res = new ClassPathResource("config/spring/factorybean.xml");
        BeanFactory factory = new XmlBeanFactory(res);
        System.out.println(factory.getBean("factoryBeanOne").getClass());
        System.out.println(factory.getBean("factoryBeanTwo").getClass());
        System.out.println(factory.getBean("&factoryBeanTwo").getClass());

    }
}
```

输出:

```java
class java.util.Date
class java.lang.String
class com.demo.factorybean.MyFactoryBean
```

注:

如果希望获取MyFactoryBean实例，则需要在getBean(beanName)方法时在beanName前显示加上"&"前缀, 例如

```
factory.getBean("&factoryBeanTwo")
```