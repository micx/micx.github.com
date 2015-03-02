---
layout: post
title: Future & FutureTask
category: java技术
published: true
---


##问题

1. 我有一个任务需要异步处理（一般情况就是创建一个线程去跑）
2. 同时又想为这个任务设定超时时间

java中有一种Future模式，可以这样描述：

我有一个任务，提交给了Future，Future替我完成这个任务，期间我自己可以去执行其他任务，一段时间之后，我就便可以从Future那儿取出结果。

类比现实中的场景就相当于下了一张订货单，一段时间后可以拿着提订单来提货，这期间可以干别的事情。

其中Future 接口就是订货单，真正处理订单的是Executor类，它根据Future接口的要求来生产产品。

Future接口提供方法来检测任务是否被执行完，等待任务执行完获得结果，也可以设置任务执行的超时时间。这个设置超时的方法就是实现Java程序执行超时的关键。

Future接口是一个泛型接口，严格的格式应该是Future<V>，其中V代表了Future执行的任务返回值的类型。 Future接口的方法介绍如下：

* <font color="#009393">_boolean cancel (boolean mayInterruptIfRunning)_</font> 取消任务的执行。参数指定是否立即中断任务执行，或者等等任务结束
* <font color="#009393">_boolean isCancelled ()_</font> 任务是否已经取消，任务正常完成前将其取消，则返回 _true_
* <font color="#009393">_boolean isDone ()_</font>任务是否已经完成。需要注意的是如果任务正常终止、异常或取消，都将返回 _true_
* <font color="#009393">_V get () throws InterruptedException, ExecutionException_</font>  等待任务执行结束，然后获得V类型的结果。_InterruptedException _ 线程被中断异常， _ExecutionException_任务执行异常，如果任务被取消，还会抛出_CancellationException_
* <font color="#009393">_V get (long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException_</font> 同上面的 _get_ 功能一样，多了设置超时时间。参数timeout指定超时时间，_uint_ 指定时间的单位，在枚举类 _TimeUnit_ 中有相关的定义。如果计算超时，将抛出 _TimeoutException_

Future的实现类有java.util.concurrent.FutureTask<V>即 javax.swing.SwingWorker<T,V>。通常使用FutureTask来处理我们的任务。FutureTask类同时又实现了Runnable接口，所以可以直接提交给Executor执行。使用FutureTask实现超时执行的代码如下：

```java
public class FutureTest {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        FutureTask<String> futureTask = new FutureTask<String>(new Callable<String>() {//使用Callable接口作为构造参数
                    public String call() throws InterruptedException {
                        int rst = 0;
                        for(int i=0;i<10000;++i){
                            rst += Math.random()*10;
                        }
                        return "future test\t"+rst;
                    }});
        long startTime = System.currentTimeMillis();
        executor.submit(futureTask);
        //other operations...
        try {
            String result = futureTask.get(20000, TimeUnit.MILLISECONDS); //取得结果，同时设置超时执行时间为5秒。同样可以用future.get()，不设置执行超时时间取得结果
            System.out.println(String.format("cost: %d ms",(System.currentTimeMillis() - startTime)));
            System.out.println(result);
        } catch (InterruptedException e) {
            futureTask.cancel(true);
            System.out.println(e);
        } catch (ExecutionException e) {
            futureTask.cancel(true);
            System.out.println(e);
        } catch (TimeoutException e) {
            futureTask.cancel(true);
            System.out.println(e);
        } finally {
            executor.shutdown();
        }
    }
}
```

执行结果:

```java
cost: 8 ms
future test	45043
```

如果将超时时间设置为5ms

```java
String result = futureTask.get(5, TimeUnit.MILLISECONDS); 

```

执行结果:

```java
java.util.concurrent.TimeoutException
```

***

参考文献：

1. [Java™ Platform Standard Ed. 6 -- Future & FutureTask](http://docs.oracle.com/javase/6/docs/api/)