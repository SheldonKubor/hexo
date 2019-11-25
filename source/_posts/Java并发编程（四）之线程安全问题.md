---
title: Java并发编程（四）之线程安全问题
date: 2019-11-24 19:24:41
tags: [多线程,并发,Java]
---

讲完创建线程的方法，接下来我们学习一下多线程编程中会遇到的线程安全问题。

谈到线程安全，就会涉及到共享资源，所谓共享资源，就是指该资源被多个线程使用。线程安全问题就是，当多个线程同时读写同一个共享资源的时候，没有加任何同步措施，导致出现脏数据，以及预料之外的结果。

例如下面代码：

```java
public class ThreadSafeTest {
    public static int count = 5;
    public static void main(String[] args) {
        MyThreadSafeTest thread1 = new MyThreadSafeTest();
        MyThreadSafeTest thread2 = new MyThreadSafeTest();
        thread1.start();
        thread2.start();
    }
}

class MyThreadSafeTest extends Thread{

    @Override
    public void run() {   
        while(ThreadSafeTest.count>0){
            try {
                Thread.sleep(500);
                sale();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
        }   
    }

    public void sale() {
        if (ThreadSafeTest.count > 0) {
            --ThreadSafeTest.count;
            System.out.println(Thread.currentThread().getName() + ",出售第" + (5 - ThreadSafeTest.count) + "张票");
        }
        
    }
}
```

```java
Thread-1,出售第2张票
Thread-0,出售第2张票
Thread-1,出售第3张票
Thread-0,出售第4张票
Thread-1,出售第5张票
```

我们模拟了多窗口售票的问题，每个线程就是一个窗口。可以看到，会出现一张票卖出多次的情况，这显然是不合理的。

这是由于什么原因导致的呢？要查找其原因，首先我们需要了解Java的内存模型。如下图：

![Java内存模型](/pic_doc/java_mem_model.jpg) 

Java内存模型规定，将所有的变量都放在主内存中，当线程使用变量的时候从主内存中将变量复制一份到自己的工作空间或者说工作内存，线程读写变量时操作的是自己工作内存中的变量。

Java内存模型是一个抽象的概念，实际实现中的工作内存是什么样子呢？

![](/pic_doc/cpu_mem_model.jpg) 

图中所示是一个双核CPU的架构，Java内存模型里面的工作内存就对应这里的L1或者L2或者CPU的寄存器。

当一个线程操作共享变量时，它首先从主内存复制共享变量到自己的工作内存，对工作内存里的变量进行处理，处理完后将变最值更新到主内存。

那么假如线程A和线程B同时处理一个共享变量，会出现什么情况？我们使用图中所示CPU架构，假设线程A和线程B使用不同CPU执行，并且当前两级Cache都为空。那么这时候由于Cache的存在，将会导致内存不可见问题，具体看下面的分析。

1. 线程A首先获取共享变量X的值，由于两级Cache都没有命中，所以加教木中X的值，假如为0。然后把X=0的值缓存到两级缓存，线程A修改X的值为1，然后将其写入两级Cache，并且刷新到主内存。线程A操作完毕后，线程A所在CPU的两级Cache内和主内存里面的X的值都是1。
2. 线程B获取X的值，首先一级缓存没有命中，然后看二级缓存，二级缓存命的所以返回X=1，到这里一切都是正常的，因为这时候主内存中也是X=1。然后线程B修改X的值为2,并将其存放线程B所在的一级Cache和共享二级Cache中，最后更新主内存中X的值为2，到这里一切都是好的。
3. 线程A这次又需要修改X的值，获取时一级缓存命中，并且X=1，到这里问题与出现了，明明线程B已经把X的值修改为了2，为何线程A获取的还是1呢？这就是共享变量的内存不可见问题，也就是线程B写入的值对线程A不可见。

了解了这些，我们就会明白，上述代码中，一张票卖多次的问题是如何导致的了，就是因为`thread1`与`thread2`由于内存模型导致的共享变量`count`内存不可见。

那么如何解决呢？

大学时候我们学过使用`synchronized`加锁来解决线程安全问题，例如上述卖票代码改为如下：

```java
public class ThreadSafeTest {
    public static int count = 5;
    public static String lock = "lock";
    public static void main(String[] args) {
        MyThreadSafeTest thread1 = new MyThreadSafeTest();
        MyThreadSafeTest thread2 = new MyThreadSafeTest();
        thread1.start();
        thread2.start();
    }
}

class MyThreadSafeTest extends Thread{

    @Override
    public void run() {   
        while(ThreadSafeTest.count>0){
            try {
                Thread.sleep(500);
                sale();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
        }   
    }

    public void sale() {
        synchronized(ThreadSafeTest.lock){
            if (ThreadSafeTest.count > 0) {
                --ThreadSafeTest.count;
                System.out.println(Thread.currentThread().getName() + ",出售第" + (5 - ThreadSafeTest.count) + "张票");
            }
        }
        
    }
}
```
```java
Thread-0,出售第1张票
Thread-1,出售第2张票
Thread-0,出售第3张票
Thread-1,出售第4张票
Thread-0,出售第5张票
```

我们对卖票的操作进行了加锁，加锁之后，就没有出现共享变量混乱的问题，那么`synchronized`到底是什么呢？怎么实现的共享变量内存可见呢？

`synchronized`块是Java提供的一种原子性内置锁，Java中的每个对象都可以把它当做一个同步锁来使用，这些Java内置的使用者看不到的锁被称为内部锁，也叫作监视器锁，线程的执行代码在进入`synchronized`代码块前会自动获取内部锁，这时候其他线程访问该同步代码块时会被阻塞挂起。拿到内部锁的线程会在正常退出同步代码块或者抛出异常后或者在同步块内调用了该内置锁资源的`wait`系列方法时释放该内置锁。内置锁是排它锁，也就是当一个线程获取这个锁后，其他线程必须等待该线程释放锁后才能获取该锁。

由于共享变量内存可见性问题主要是由于线程的工作内存导致的，使用`synchronized`的时候，它的内存语义是什么样的呢？（也就是加锁和释放锁的语义）。

进入`synchronized`块的内存语义（加锁）是把在`synchronized`共内使用到的共享变量从线程的工作内存中清除，这样在`synchronized`块内使用到该变量时就不会从线程的工作内存中获取，而是直接从主内存中获取。退出`synchronized`块的内存语义（释放锁）是把在`synchronized`块内对共享变量的修改刷新到主内存。

除可以解决共享变量内存可见性问题外，synchronized 经常被用来实现原子性操作。另外请注意，synchronized 关键字会引起线程上下文切换并带来线程调度开销，所以使用锁太笨重，尤其是在Java1.6之前，没有对synchronized进行优化。对于内存可见性的问题，Java还提供了一种弱同步，也就是使用 `volatile` 关键字，这个由于篇幅问题，下一篇文章再讲吧。


