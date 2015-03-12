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


##第6条 消除过期的对象引用

考虑如下例子：

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack(){
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    
    public void push(Object obj){
        elements[size++] = obj;
    }
    
    public Object pop(){
        if(size == 0){
            throw new EmptyStackException();
        }
        return elements[--size];
    }
    
    private void ensureCapacity() {
        if(elements.length == size){
            elements = Arrays.copyOf(elements, 2*size+1);
        }
    }
}
```

上述代码没有明显错误，但是隐藏着一个***"内存泄露"***的问题，如果一个栈先是增长，然后收缩，那么从栈中弹出来的对象将不会被当做垃圾回收，即使使用栈的程序不再引用这些对象，它们也不会被回收。

在支持垃圾回收的语言中，内存泄露是很隐蔽的（这类内存泄露称为"***无意识的对象保持(unintentional object retention)***"）

这类问题的修复方法很简单：***一旦对象引用已经过期，只需清空这些引用即可***。

```java
public Object pop(){
    if(size == 0){
        throw new EmptyStackException();
    }
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```

清空引用的另一个好处是，如果它们以后又被错误的解除引用，程序就会抛出NPE异常，而不是悄悄的错误运行下去。

一般而言，常见内存泄露：

* 只要是程序员自己管理的内存，就应该警惕内存泄露问题。一旦元素被释放，则该元素中包含的任何对象引用都应该被清空。

* 内存泄露另一个常见来源是缓存。

* 第三个常见来源是监听器和其他回调。

内存泄露分析工具(Heap Profiler)



##第7条 避免使用终结方法

***终结方法(finalizer)通常是不可预测的，也少很危险的，一般情况下是不必要的***。终结方法的缺点在于不能保证会被及时的执行[[JLS, 12.6](http://docs.oracle.com/javase/specs/jls/se7/html/jls-12.html#jls-12.6)]。从一个对象变得不可到达开始，到它的终结方法被执行，所花费的这段时间是任意长的。这意味着对时间性有要求的任务不应该由终结方法执行，例如用终结方法来关闭已经打开的文件，这是严重错误的，因为打开文件的描述符是一种很有限的资源，由于JVM会延迟执行终结方法，所以大量的文件会保留打开状态。

终结方法的线程的优先级一般比应用程序的其他线程低的多。

java语言规范不仅不保证终结方法会被及时地执行，而且根本就不保证它们会被执行，当一个程序终止的时候，某些已经无法访问的对象上的终结方法却根本没有被执行，这完全是有可能的。

***结论：不应该依赖终结方法来更新重要的持久状态***。

例如，依赖终结方法来释放共享资源(比如数据库)上的永久锁，很容易让整个分布式系统垮掉。

总之，除非是作为安全网，或者是为了终止非关键的本地资源，否则不要使用终结方法。在这些很少见的情况下，既然使用了终结方法，就要记住调用super.finalize。如果终结方法作为安全网，要记得记录终结方法的非法用法，最后，如果需要把终结方法与公有的非final类关联起来，请考虑使用终结方法守卫者，以确保即使子类的终结方法未能调用super.finalize，该终结方法也会被执行。

***

##第9条 覆盖equals时总要覆盖hashCode

***在每个覆盖了equals方法的类中，必须覆盖hashCode方法***。如果不这样做，就会违反Object.hashCode的通用约定，从而导致该类无法结合所有 ***基于散列的集合*** 一起正常运作（HashMap、HashSet、HashTable）

Object.hashCode通用约定，摘自Object规范[JavaSE6]:

* 在应用程序执行期间，只要对象的equals方法的比较操作所用到的信息没有被修改，那么对同一个对象调用多次，hashCode方法都必须始终如一的返回同一个整数。

* 如果两个对象根据equals(Object)方法比较是相等的，那么调用这两个对象中任意一个对象的hashCode方法都必须产生相同的整数结果。

* 如果两个对象根据equals(Object)方法比较是不相等的，那么这两个对象的hashCode值不一定不相等。


*因没有覆盖hashCode而违反的关键约定是第二条：相等的对象必须具有相等的散列码(hash code)*。

***相等的对象必须有相等的散列码(hash code)***，一个好的散列函数通常倾向于"为不相等的对象产生不相等的散列码"，理想情况下，散列函数应该把集合中不相等的实例均匀地分布到所有可能的散列值上。

如果一个类是不可变的，并且计算散列码的开销也比较大，就应该考虑 ***把散列码缓存在对象内部*** ，而不是每次请求时都重新计算散列码。例如String的HashCode()实现:

```java

/** The value is used for character storage. */
private final char value[];

/** The offset is the first index of the storage that is used. */
private final int offset;

/** The count is the number of characters in the String. */
private final int count;

/** Cache the hash code for the string */
private int hash; // Default to 0

...

/**
 * Returns a hash code for this string. The hash code for a
 * <code>String</code> object is computed as
 * <blockquote><pre>
 * s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
 * </pre></blockquote>
 * using <code>int</code> arithmetic, where <code>s[i]</code> is the
 * <i>i</i>th character of the string, <code>n</code> is the length of
 * the string, and <code>^</code> indicates exponentiation.
 * (The hash value of the empty string is zero.)
 *
 * @return  a hash code value for this object.
 */
public int hashCode() {
  int h = hash;
  int len = count;
  if (h == 0 && len > 0) {
      int off = offset;
      char val[] = value;
      for (int i = 0; i < len; i++) {
          h = 31*h + val[off++];
      }
      hash = h;
  }
  return h;
}
```

注意： ***不要试图从散列码计算中排除掉一个对象的关键部分来提高性能***

***

##第10条 始终覆盖toString

虽然java.lang.Object提供了toString方法的一个实现，但它返回的字符串通常不是类的用户所希望看到的。它包含类的名称，以及一个"@"符号，接着是散列码的无符号十六进制表示法，如"Hello@163b12"。

```java
public String toString() {
  return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```

覆盖toString方法并不像遵守equals和hashCode（第9条）那么重要，但是，提供好的toString实现可以使类用起来更加舒适。

toString方法应该返回对象中包含的所有值得关注的信息。如果对象太大，或者对象中包含的状态信息难以用字符串来表达，这种情况下，toString应该返回一个摘要信息。

理想情况下，字符串应该是自描述的(self-explanatory)（Thread例子不满足这样的要求）。

一种比较方便通用的方法是使用`org.apache.commons.lang3.builder.ToStringBuilder`

```java
@Override
public String toString() {
    return ToStringBuilder.reflectionToString(this,
            ToStringStyle.SHORT_PREFIX_STYLE);
}
```

## 第12条 考虑实现Comparable接口

为实现Comparable接口的对象进行排序：

```java
Arrays.sort(a);
```

强烈建议(x.compareTo(y) == 0) == (x.equals(y)), 但这并非绝对必要。一般来说任何实现了Comparable接口的类，若违反了这个条件，都应该明确予以说明。例如：“注意：该类具有内在的排序功能，但是与equals不一致” 。这条建议并不是真正的规则，只是说明了compareTo方法施加的等同性测试，在通常情况下应该返回与equals方法相同的结果。

例如，BigDecimal类：

```java

public class CompareToTest {
    public static void main(String[] args) {

        BigDecimal one = new BigDecimal("1.0");
        BigDecimal two = new BigDecimal("1.00");

        Set<BigDecimal> hashSet = new HashSet<BigDecimal>();
        Set<BigDecimal> treeSet = new TreeSet<BigDecimal>();
       
        hashSet.add(one);
        hashSet.add(two);
        treeSet.add(one);
        treeSet.add(two);
       

        System.out.println("compare:\t" + one.compareTo(two));
        System.out.println("equals:\t" + one.equals(two));
        System.out.println("hashSet:\t" + hashSet.size());
        System.out.println("treeSet:\t" + treeSet.size());

    }
}

```

输出结果：

```java
compare:    0
equals: false
hashSet:    2
treeSet:    1
```

这里HashSet使用的是equals方法，而TreeSet使用的是Compare方法。

来看一下BigDecimal类的equals方法

```java
 /**
     * Compares this {@code BigDecimal} with the specified
     * {@code Object} for equality.  Unlike {@link
     * #compareTo(BigDecimal) compareTo}, this method considers two
     * {@code BigDecimal} objects equal only if they are equal in
     * value and scale (thus 2.0 is not equal to 2.00 when compared by
     * this method).
     *
     * @param  x {@code Object} to which this {@code BigDecimal} is 
     *         to be compared.
     * @return {@code true} if and only if the specified {@code Object} is a
     *         {@code BigDecimal} whose value and scale are equal to this 
     *         {@code BigDecimal}'s.
     * @see    #compareTo(java.math.BigDecimal)
     * @see    #hashCode
     */
    @Override
    public boolean equals(Object x) {
        if (!(x instanceof BigDecimal))
            return false;
        BigDecimal xDec = (BigDecimal) x;
        if (x == this)
            return true;
    if (scale != xDec.scale)
        return false;
        long s = this.intCompact;
        long xs = xDec.intCompact;
        if (s != INFLATED) {
            if (xs == INFLATED)
                xs = compactValFor(xDec.intVal);
            return xs == s;
        } else if (xs != INFLATED)
            return xs == compactValFor(this.intVal);
        
        return this.inflate().equals(xDec.inflate());
    }
```




参考文献：

1. Effect Java

2. [The Java® Language Specification](http://docs.oracle.com/javase/specs/jls/se7/html/)


