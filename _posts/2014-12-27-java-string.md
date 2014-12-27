---
layout: post
title: String & StringBuffer & StringBuilder
category: java技术
published: true
---


## 概述
* String 字符串常量
* StringBuffer 字符串变量（线程安全）
* StringBuilder 字符串变量（非线程安全）

## 性能 

StringBuilder > StringBuffer > String

## String

String 类型和 StringBuffer 类型的主要性能区别其实在于 String 是不可变的对象, 因此在每次对 String 类型进行改变的时候其实都等同于生成了一个新的 String 对象，然后将指针指向新的 String 对象，所以经常改变内容的字符串最好不要用 String ，因为每次生成对象都会对系统性能产生影响，特别当内存中无引用对象多了以后， JVM 的 GC 就会开始工作，那速度是一定会相当慢的。

 而如果是使用 StringBuffer 类则结果就不一样了，每次结果都会对 StringBuffer 对象本身进行操作，而不是生成新的对象，再改变对象引用。所以在一般情况下我们推荐使用 StringBuffer ，特别是字符串对象经常改变的情况下。


##StringBuffer

StringBuffer线程安全的可变字符序列。一个类似于 String 的字符串缓冲区，但不能修改。虽然在任意时间点上它都包含某种特定的字符序列，但通过某些方法调用可以改变该序列的长度和内容。
可将字符串缓冲区安全地用于多个线程。可以在必要时对这些方法进行同步，因此任意特定实例上的所有操作就好像是以串行顺序发生的，该顺序与所涉及的每个线程进行的方法调用顺序一致。

StringBuffer 上的主要操作是 append 和 insert 方法，可重载这些方法，以接受任意类型的数据。每个方法都能有效地将给定的数据转换成字符串，然后将该字符串的字符追加或插入到字符串缓冲区中。append 方法始终将这些字符添加到缓冲区的末端；而 insert 方法则在指定的点添加字符。
例如，如果 z 引用一个当前内容是“start”的字符串缓冲区对象，则此方法调用 z.append("le") 会使字符串缓冲区包含“startle”，而 z.insert(4, "le") 将更改字符串缓冲区，使之包含“starlet”。
在大部分情况下 StringBuilder > StringBuffer

##StringBuilder

java.lang.StringBuilder一个可变的字符序列是5.0新增的。此类提供一个与 StringBuffer 兼容的 API，但不保证同步。该类被设计用作 StringBuffer 的一个简易替换，用在字符串缓冲区被单个线程使用的时候（这种情况很普遍）。如果可能，建议优先采用该类，因为在大多数实现中，它比 StringBuffer 要快。两者的方法基本相同。

## 测试

```java
package com.demo.string;

import com.google.common.base.Charsets;

import java.util.Random;

/**
 * Created by micx  on 2014/12/27 下午5:03.
 */
public class StringTest {
    private static final Random random = new Random();
    private static final int len = 100000; 
    
    public static void main(String[] args) {
        StringBuilder sb = new StringBuilder();
        for(int i=0;i<len;++i){
            sb.append(getRandomChar());
        }
        String str = sb.toString();
        stringTest(str);
        stringBufferTest(str);
        stringBuilderTest(str);

    }
    private static String stringTest(String str) {
        long start = System.currentTimeMillis();
        byte[] bytes = str.getBytes(Charsets.UTF_8);
        String ret ="";
        for(int i=0;i<bytes.length;++i){
            ret += (char)bytes[i];
        }
        long cost = System.currentTimeMillis() - start;
        System.out.println("String cost:\t" + cost+"ms");
        return ret;
    }

    private static String stringBufferTest(String str) {
        long start = System.currentTimeMillis();
        byte[] bytes = str.getBytes(Charsets.UTF_8);
        StringBuffer ret = new StringBuffer();
        for(int i=0;i<bytes.length;++i){
            ret.append((char)bytes[i]);
        }
        long cost = System.currentTimeMillis() - start;
        System.out.println("StringBuffer cost:\t" + cost+"ms");
        return ret.toString();
    }
    private static String stringBuilderTest(String str) {
        long start = System.currentTimeMillis();
        byte[] bytes = str.getBytes(Charsets.UTF_8);
        StringBuilder ret = new StringBuilder();
        for(int i=0;i<bytes.length;++i){
            ret.append((char)bytes[i]);
        }
        long cost = System.currentTimeMillis() - start;
        System.out.println("StringBuilder cost:\t" + cost+"ms");
        return ret.toString();
    }
    
    private static char getRandomChar(){
        return (char) random.nextLong();
    }
}
```

运行结果：

```java
String cost:	75011ms
StringBuffer cost:	21ms
StringBuilder cost:	17ms
```



