---
layout: post
title: Effect Java 读书笔记
category: java技术
published: true
---


##第5条 避免创建不必要的对象

重用对象而不是在每次需要的时候创建一个相同功能的新对象。

反面例子：

```java
String s = new String("string test");

```

该语句每次被执行时都创建一个新String实例，传递给String构造器的参数("string test")本身就是一个String实例，如果这种用法在一个循环或者频繁调用的方法中，就会创建出成千上万不必要的String实例。

改进版:

```java
String s = "string test";
```

该版本只用了一个String实例，而不是每次执行都创建一个新的实例。 而且，它可以保证，对于所有在同一台虚拟机中运行的代码，只要它们包含相同的字符串字面常量，该对象就会被重用[[JLS, 3.10.5](http://docs.oracle.com/javase/specs/jls/se7/html/jls-3.html#jls-3.10.5)]

```java
class Test {
    public static void main(String[] args) {
        String hello = "Hello", lo = "lo";
        System.out.print((hello == "Hello") + " ");
        System.out.print((Other.hello == hello) + " ");
        System.out.print((other.Other.hello == hello) + " ");
        System.out.print((hello == ("Hel"+"lo")) + " ");
        System.out.print((hello == ("Hel"+lo)) + " ");
        System.out.println(hello == ("Hel"+lo).intern());
    }
}
class Other { static String hello = "Hello"; }
```

```java
true true true true false true
```

对于同时提供了静态工厂方法和构造器的不可变类，通常使用静态工厂方法而不是构造器，以避免创建不必要的对象。例如，静态工厂方法

```java
Boolean.valueOf(String)
```

几乎总是优先于构造器

```java
Boolean(String)
```

***优先使用基本类型而不是装箱类型，当心无意识的自动装箱***

```java
public class SumTest {
    public static void main(String[] args) {
        sum_1();
        sum_2();
    }

    private static void sum_2() {
        long start = System.currentTimeMillis();
        long sum = 0L;
        for (long i = 0; i < Integer.MAX_VALUE; i++) {
            sum += i;
        }
        long cost = System.currentTimeMillis() - start;
        System.out.println("sum-2 cost: " + cost + "ms");
        System.out.println(sum);
    }

    private static void sum_1() {
        long start = System.currentTimeMillis();
        Long sum = 0L;
        for (long i = 0; i < Integer.MAX_VALUE; i++) {
            sum += i;
        }
        long cost = System.currentTimeMillis() - start;
        System.out.println("sum-1 cost: " + cost + "ms");
        System.out.println(sum);
    }
}
```

执行结果:

```java
sum-1 cost: 9595ms
2305843005992468481
sum-2 cost: 1495ms
2305843005992468481
```

同时，通过维护自己的***对象池(object pool)***来避免创建对象并不是一种好的做法，除非池中的***对象非常重量级***。真正正确使用对象池的典型对象示例就是***数据库连接池***。现代的JVM实现具有高度优化的垃圾回收器，其性能很容易超过轻量级的对象池的性能。



***

参考文献：

1. Effect Java

2. [The Java® Language Specification](http://docs.oracle.com/javase/specs/jls/se7/html/)


