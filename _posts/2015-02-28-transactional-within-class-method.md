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

这里我们首先来看一下[Spring @Transactional是如何工作的？](http://micx.github.io/java%E6%8A%80%E6%9C%AF/2015/02/27/spring-transactional.html)<sup>[1]<sup>
	
声明式事务管理主要包含三个组成部分：

* 事务的切面
* EntityManager Proxy本身
* 事务管理器

问题就在于Spring的事务是通过切面实现的，而[Spring AOP是通过代理对象实现的(详见 6.6.1)](http://docs.spring.io/spring/docs/2.5.x/reference/aop.html)<sup>[2]<sup>

因此，假设对于代理对象bean有如下的调用堆栈： 

bean.A() -> bean.B() -> bean.C() -> ... 

那么只有第一个bean.A()的调用是在代理对象上调用的，而且后续对B/C/...方法的调用全部是在原始对象上调用。即在Bean类的内部，所看到的this对象都是未增强的对象，也就是这个this是真正的this对象，这一点看似正常。 

同样的道理，假设我们调用是：

bean.B() -> bean.C() -> ... 

那么这时的bean.B()调用又是被增强的。

所以回到刚开始的问题，Spring @Transactional - 方法a()调用事务方法b()，方法b()就不再是代理对象调用，而是有this对象调用，方法b()就得到增强，进而事务注解无法生效。


参考文献：

1. [How does Spring @Transactional Really Work?](http://www.javacodegeeks.com/2014/06/how-does-spring-transactional-really-work.html)

2. [Aspect Oriented Programming with Spring](http://docs.spring.io/spring/docs/2.5.x/reference/aop.html)