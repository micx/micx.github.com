---
layout: post
title: Spring 方法内事务问题
category: java技术
published: true
---


##1. 问题

Spring @Transactional - 方法内调用事务方法，事务未生效

```java
public class TransactionTest{
	
	public void a(){
		b();
	}

	@Transactional
	public void b(){
		//Transaction operations ...
	}

}
```

##2. @Transactional是如何工作的 <sup>[1]<sup>
	


声明式事务管理主要包含三个组成部分：

* 事务的切面
* EntityManager Proxy本身
* 事务管理器

###2.1 事务的切面

事务的切面是一个“around（环绕）”切面，在注解的业务方法前后都可以被调用。实现切面的具体类是TransactionInterceptor。

事务的切面有两个主要职责：

* 在’before’时，切面提供一个调用点，来决定被调用业务方法应该在正在进行事务的范围内运行，还是开始一个新的独立事务。
* 在’after’时，切面需要确定事务被提交，回滚或者继续运行。

在’before’时，事务切面自身不包含任何决策逻辑，是否开始新事务的决策委派给事务管理器完成。

###2.2 EntityManager proxy

当业务方法调用entityManager.persist()时，这不是由entity manager直接调用的，而是业务方法调用代理，代理从线程获取当前的entity manager，事务管理器将entity manager绑定到线程。

了解了@Transactional机制的各个部分，我们来看一下实现它的常用Spring配置。

###2.3 事务管理器

事务管理器需要解决下面两个问题：

* 新的Entity Manager是否应该被创建？
* 是否应该开始新的事务？

这些需要事务切面’before’逻辑被调用时决定。

事务管理器的决策基于以下两点：

* 事务是否正在进行
* 事务方法的propagation属性（比如REQUIRES_NEW总要开始新事务）

如果事务管理器确定要创建新事务，那么将：

* 创建一个新的entity manager
* entity manager绑定到当前线程
* 从数据库连接池中获取连接
* 将连接绑定到当前线程

使用ThreadLocal变量将entity manager和数据库连接都绑定到当前线程。

事务运行时他们存储在线程中，当它们不再被使用时，事务管理器决定是否将他们清除。

程序的任何部分如果需要当前的entity manager和数据库连接都可以从线程中获取。

###2.4 整合三个部分

如何将三个部分组合起来使事务注解可以正确地发挥作用呢？首先定义entity manager工厂。

这样就可以通过持久化上下文注解注入Entity Manager proxy。

```java
@Configuration
public class EntityManagerFactoriesConfiguration {
    @Autowired
    private DataSource dataSource;

    @Bean(name = "entityManagerFactory")
    public LocalContainerEntityManagerFactoryBean emf() {
        LocalContainerEntityManagerFactoryBean emf = ...
        emf.setDataSource(dataSource);
        emf.setPackagesToScan(
            new String[] {"your.package"});
        emf.setJpaVendorAdapter(
            new HibernateJpaVendorAdapter());
        return emf;
    }
}
```

下一步实现配置事务管理器和在@Transactional注解的类中应用事务的切面。

```java
@Configuration
@EnableTransactionManagement
public class TransactionManagersConfig {
    @Autowired
    EntityManagerFactory emf;
    @Autowired
    private DataSource dataSource;

    @Bean(name = "transactionManager")
    public PlatformTransactionManager transactionManager() {
        JpaTransactionManager tm =
            new JpaTransactionManager();
            tm.setEntityManagerFactory(emf);
            tm.setDataSource(dataSource);
        return tm;
    }
}
```

注解@EnableTransactionManagement通知Spring，@Transactional注解的类被事务的切面包围。这样@Transactional就可以使用了。

总结

Spring声明式事务管理机制非常强大，但它可能被误用或者容易发生配置错误。

当这个机制不能正常工作或者未达到预期运行结果等问题出现时，理解它的内部工作情况是很有帮助的。

需要记住的最重要的一点是，要考虑到两个概念：事务和持久化上下文，每个都有自己不可读的明显的生命周期。


参考文献：

1. [How does Spring @Transactional Really Work?](http://www.javacodegeeks.com/2014/06/how-does-spring-transactional-really-work.html)