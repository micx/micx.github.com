---
---
layout: post
title: SimpleDateFormat的线程安全问题与解决方案
category: java技术
published: true
---
## 1. 原因

SimpleDateFormat(下面简称sdf)类内部有一个Calendar对象引用,它用来储存和这个sdf相关的日期信息,例如sdf.parse(dateStr), sdf.format(date) 诸如此类的方法参数传入的日期相关String, Date等等, 都是交友Calendar引用来储存的.这样就会导致一个问题,如果你的sdf是个static的, 那么多个thread 之间就会共享这个sdf, 同时也是共享这个Calendar引用, 并且, 观察 sdf.parse() 方法,你会发现有如下的调用:

```java
Date parse() {
  calendar.clear(); // 清理calendar
  ... // 执行一些操作, 设置 calendar 的日期什么的
  calendar.getTime(); // 获取calendar的时间
}
```

这里会导致的问题就是, 如果 线程A 调用了 sdf.parse(), 并且进行了 calendar.clear()后还未执行calendar.getTime()的时候,线程B又调用了sdf.parse(), 这时候线程B也执行了sdf.clear()方法, 这样就导致线程A的的calendar数据被清空了(实际上A,B的同时被清空了). 又或者当 A 执行了calendar.clear() 后被挂起, 这时候B 开始调用sdf.parse()并顺利i结束, 这样 A 的 calendar内存储的的date 变成了后来B设置的calendar的date

## 2. 问题重现

```java
public class DateFormatTest extends Thread {
    private static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

    private String name;
    private String dateStr;
    private boolean sleep;

    public DateFormatTest(String name, String dateStr, boolean sleep) {
        this.name = name;
        this.dateStr = dateStr;
        this.sleep = sleep;
    }

    @Override
    public void run() {

        Date date = null;

        if (sleep) {
            try {
                TimeUnit.MILLISECONDS.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        try {
            date = sdf.parse(dateStr);
        } catch (ParseException e) {
            e.printStackTrace();
        }

        System.out.println(name + " : date: " + date);
    }

    public static void main(String[] args) throws InterruptedException {

        ExecutorService executor = Executors.newCachedThreadPool();

        // A 会sleep 2s 后开始执行sdf.parse()
        executor.execute(new DateFormatTest("A", "1991-09-13", true));
        // B 打了断点,会卡在方法中间
        executor.execute(new DateFormatTest("B", "2013-09-13", false));

        executor.shutdown();
    }
}

```

使用Debug模式执行这段代码,并在sdf.parse()方法里打上断点

```java
parse() {
    calendar.clear()

    // 这里打一个断点
    
    calendar.getTime()
}
```
过程:

1) 首先A线程跑起来以后会进入sleep

2) B线程跑起来, 卡在断点处

3) A线程醒过来, 执行 calendar.clear(), 并将设置sdf.calendar的date为1991-09-13, 此时 A B 的 calendar 都为 1991-09-13

4) 让断点继续执行, 输出如下

```java
A : date: Fri Sep 13 00:00:00 CDT 1991
B : date: Fri Sep 13 00:00:00 CDT 1991
```

## 3. 解决方案

最简单的解决方案我们可以把static去掉,这样每个新的线程都会有一个自己的sdf实例,从而避免线程安全的问题

然而,使用这种方法,在高并发的情况下会大量的new sdf以及销毁sdf,这样是非常耗费资源的

在并发情况下,网站的请求任务与线程执行情况大概可以理解为如下



例如Tomcat的线程池的最大Thread数为4, 现在需要执行的任务有1000个(理解为有1000个用户点了你的网站的某个功能),

而这1000个任务都会用到我们写的日期函数处理类

A) 假如说日期函数处理类使用的是new SimpleDateFormat的方法,那么这里就会有1000次sdf的创建和销毁

B) Java中提供了一种ThreadLocal的解决方案,它的工作方式是,每个线程只会有一个实例,也就是说我们执行完这1000个任务,总共只会实例化4个sdf.

而且,它并不会有多线程的并发问题,因为,单个线程执行任务肯定是顺序的,例如Thread #1负责执行Task #1-#250, 那么他是顺序而执行Task #1-#250

而Thread #2拥有自己的sdf实例,他也是顺序执行任务 Task #251-#500, 以此类推

下面是一个使用ThreadLocal解决sdf多线程问题的例子

```java
public class DateUtil {

    /** 锁对象 */
    private static final Object lockObj = new Object();

    /** 存放不同的日期模板格式的sdf的Map */
    private static Map<String, ThreadLocal<SimpleDateFormat>> sdfMap = new HashMap<String, ThreadLocal<SimpleDateFormat>>();

    /**
     * 返回一个ThreadLocal的sdf,每个线程只会new一次sdf
     * 
     * @param pattern
     * @return
     */
    private static SimpleDateFormat getSdf(final String pattern) {
        ThreadLocal<SimpleDateFormat> tl = sdfMap.get(pattern);

        // 此处的双重判断和同步是为了防止sdfMap这个单例被多次put重复的sdf
        if (tl == null) {
            synchronized (lockObj) {
                tl = sdfMap.get(pattern);
                if (tl == null) {
                    // 只有Map中还没有这个pattern的sdf才会生成新的sdf并放入map
                    System.out.println("put new sdf of pattern " + pattern + " to map");

                    // 这里是关键,使用ThreadLocal<SimpleDateFormat>替代原来直接new SimpleDateFormat
                    tl = new ThreadLocal<SimpleDateFormat>() {

                        @Override
                        protected SimpleDateFormat initialValue() {
                            System.out.println("thread: " + Thread.currentThread() + " init pattern: " + pattern);
                            return new SimpleDateFormat(pattern);
                        }
                    };
                    sdfMap.put(pattern, tl);
                }
            }
        }

        return tl.get();
    }

    /**
     * 是用ThreadLocal<SimpleDateFormat>来获取SimpleDateFormat,这样每个线程只会有一个SimpleDateFormat
     * 
     * @param date
     * @param pattern
     * @return
     */
    public static String format(Date date, String pattern) {
        return getSdf(pattern).format(date);
    }

    public static Date parse(String dateStr, String pattern) throws ParseException {
        return getSdf(pattern).parse(dateStr);
    }

}
```

测试类

```java
public class Test {

    public static void main(String[] args) {
        final String patten1 = "yyyy-MM-dd";
        final String patten2 = "yyyy-MM";

        Thread t1 = new Thread() {

            @Override
            public void run() {
                try {
                    DateUtil.parse("1992-09-13", patten1);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        };

        Thread t2 = new Thread() {

            @Override
            public void run() {
                try {
                    DateUtil.parse("2000-09", patten2);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        };

        Thread t3 = new Thread() {

            @Override
            public void run() {
                try {
                    DateUtil.parse("1992-09-13", patten1);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        };

        Thread t4 = new Thread() {

            @Override
            public void run() {
                try {
                    DateUtil.parse("2000-09", patten2);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        };

        Thread t5 = new Thread() {

            @Override
            public void run() {
                try {
                    DateUtil.parse("2000-09-13", patten1);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        };

        Thread t6 = new Thread() {

            @Override
            public void run() {
                try {
                    DateUtil.parse("2000-09", patten2);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        };

        System.out.println("单线程执行: ");
        ExecutorService exec = Executors.newFixedThreadPool(1);
        exec.execute(t1);
        exec.execute(t2);
        exec.execute(t3);
        exec.execute(t4);
        exec.execute(t5);
        exec.execute(t6);
        exec.shutdown();

        sleep(1000);

        System.out.println("双线程执行: ");
        ExecutorService exec2 = Executors.newFixedThreadPool(2);
        exec2.execute(t1);
        exec2.execute(t2);
        exec2.execute(t3);
        exec2.execute(t4);
        exec2.execute(t5);
        exec2.execute(t6);
        exec2.shutdown();
    }

    private static void sleep(long millSec) {
        try {
            TimeUnit.MILLISECONDS.sleep(millSec);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

输出

```java
单线程执行: 
put new sdf of pattern yyyy-MM-dd to map
thread: Thread[pool-1-thread-1,5,main] init pattern: yyyy-MM-dd
put new sdf of pattern yyyy-MM to map
thread: Thread[pool-1-thread-1,5,main] init pattern: yyyy-MM
双线程执行: 
thread: Thread[pool-2-thread-1,5,main] init pattern: yyyy-MM-dd
thread: Thread[pool-2-thread-2,5,main] init pattern: yyyy-MM
thread: Thread[pool-2-thread-1,5,main] init pattern: yyyy-MM
thread: Thread[pool-2-thread-2,5,main] init pattern: yyyy-MM-dd
```

从输出我们可以看出:

1) 1个线程执行这6个任务的时候,这个线程首次使用过的时候会new一个新的sdf,并且以后都一直用这个sdf,而不是每次处理任务都新建一个新的sdf

2) 2个线程执行6个任务的时候也是同理,但是2个线程的sdf是分开的,每个线程都有自己的"yyyy-MM-dd", "yyyy-MM"的sdf,所以他们不会有线程安全安全问题

试想,如果使用的是new的实现方法,那么不管是用1个线程去执行,还是用2个线程去执行这6个任务,都需要new 6个sdf