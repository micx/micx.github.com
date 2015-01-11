---
layout: post
title: tomcat 
category: java技术
published: true
---
## 1. Tomcat SEVERE: Error listenerStart 错误

Tomcat报的错比较含糊，只提示了Error listenerStart，造成的原因可能很多。

为了调试，我们要获得更详细的日志。可以在WEB-INF/classes目录下新建一个文件叫logging.properties，内容如下：


```java
handlers = org.apache.juli.FileHandler, java.util.logging.ConsoleHandler  
  
############################################################  
# Handler specific properties.  
# Describes specific configuration info for Handlers.  
############################################################  
  
org.apache.juli.FileHandler.level = FINE  
org.apache.juli.FileHandler.directory = ${catalina.base}/logs  
org.apache.juli.FileHandler.prefix = error-debug.  
  
java.util.logging.ConsoleHandler.level = FINE  
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
```

这样，我们再启动tomcat时，就会在logs目录下生成一个更详细的日志error-debug.yyyy-MM-dd.log。 

