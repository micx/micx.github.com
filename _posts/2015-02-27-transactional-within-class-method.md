---
layout: post
title: Spring 方法内事务问题
category: java技术
published: true
---


##1. 问题

方法内调用Spring @Transactional 事务未能生效

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


