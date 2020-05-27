---
title: Java并发编程（八）之CAS操作
date: 2020-04-22 10:17:18
tags: [多线程,并发,Java]
---

学习`CAS`之前，先了解一下什么是原子性操作，所谓的原子性操作，是指进行一系列操作的时候，这一系列操作要么全部执行，要么全部不执行，不会出现只执行一部分的情况。

例如，我们对一个变量进行+1（例如`x++`）操作的时候，首先要读取当前变量值，然后+1，然后再将新值赋值给该变量。这就是一个`读-改-写`的过程。如果这个过程不是原子性的，那么久就会出现线程安全问题。虽然`x++`只有一行代码，但是其中的操作并不是一步而成的，其操作不是原子性的。

通过之前所学，我们知道`synchronized`关键字可以解决线程安全问题，也就是`内存可见性`与`原子性`。但是`synchronized`是独占锁，没有获取到锁的线程将会被阻塞，会降低程序性能。例如下面代码：

```java
public class ThreadSafeCount{
    private int value;

    public synchronized int getCount(){
        return value;
    }

    public synchronized void incCount(){
        value++;
    }
}
```

可以看到两个方法都加了锁，但是同一时间只能有一个线程来调用，其中`getCount`方法是只读方法，多个线程同时调用不会有线程安全问题，但是加了synchronized关键字之后，同一时间只能有一个线程来调用，降低了并发性能。那能不能去掉这个关键字呢，也不能去掉，因为我们还需要synchronized来保证内存可见性。

### CAS操作

我们学过两个关键字`synchronized`与`volatile`，两者都可以解决共享变量内存可见性的问题，但是，`volatile`不能解决`读-改-写`等原子性的问题，`synchronized`能够解决原子性问题，但是其性能开销大。

CAS操作是`Compare And Swap`，是JDK提供的非阻塞原子性操作，通过硬件保证了`比较-更新`操作的原子性。JDK里面的`Unsafe`类提供了一系列`compareAndSwap*`方法，比如`compareAndSwapInt`和`compareAndSwapLong`。下面我们以`compareAndSwapLong`为例子进行简单介绍。

`boolean compareAndSwapLong(Object obj,long valueOffset,long expect,long update)`方法：四个参数，含义分别是：对象内存位置，对象中变量的偏移量，变量的预期值，新的值。其操作的含义是，若对象`obj`中内存偏移量为`valueOffset`的变量值为`expect`，则用新的值`update`替换旧的值，也就是替换`expect`。这是处理器提供的一个原子性命令。

CAS操作虽然很高效地解决了原子操作，但是仍然存在三大问题：`ABA问题`、`循环开销时间大`、`只能保证一个共享变量的原子操作`

1. ABA问题
CAS在操作值得时候，会检查值有没有发生变化，如果没有发生变化就更新。若是一个值原来是A，变成了B，后来又变回了A（举例：线程1获取变量值为A后，在执行CAS之前，线程2使用CAS修改变量的值为B，然后又修改变量值为A），那么使用CAS操作时检查其值得结果是没有变化，但其实是变化了，虽然线程1执行CAS操作时变量的值为A，但是这个A已经不是线程1获取变量值时的A了，这就是ABA问题。

看起来好像最后还是正确的修改了变量值，但其实是丢失了变量值得修改记录。此问题的解决方案就是使用版本号，在变量前面追加上版本号，每次变量更新的时候把版本号+1，这样`A-B-A`就会变为`1A-2B-3A`。从Java1.5开始，JDK的Atomic包里就提供了一个类`AtomicStampedReference`来解决ABA问题，其中`compareAndSet`方法的作用就是，先检查当前引用是否等于预期引用，并检查当前标志是否等于预期标志，如果全部相等，就进行更新。
```java
public boolean compareAndSet(V expectedReference,//预期引用
                                V newReference,//更新后引用
                                int expectedStamp,//当前标志
                                int newStamp)//更新后标志
```
2. 循环时间长，开销大
自旋CAS如果长时间不成功，会给CPU带来非常大的开销。

3. 只能保证一个共享变量的原子操作
当对一个共享变量执行操作的时候，我们可以使用CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。从Java1.5开始，JDK的Atomic包里就提供了一个类`AtomicReference`类来保证引用对象直接的原子性，就可以多个变量放在一个对象里来进行CAS操作。