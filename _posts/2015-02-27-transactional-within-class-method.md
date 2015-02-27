---
layout: post
title: git-多账号配置
category: linux
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


