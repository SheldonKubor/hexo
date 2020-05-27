---
title: Java并发编程（五）之线程安全问题-volatitle关键字
date: 2019-12-14 19:36:40
tags: [多线程,并发,Java]
---

上一篇提到了共享变量的线程安全问题，我们使用`synchronized`关键字来保证了共享变量的线程安全，这次我们来看看`volatitle`关键字。

当一个变量被声明为`volatitle`时，线程在写入变量的时候就不会把值缓存在寄存器或者一级二级缓存（Java内存模型中的工作内存），而是直接刷新回主内存，当其他线程读取该共享变量时，会从主内存直接获取最新的值，而不是从自己的工作内存中获取变量值。

我们看看如下代码

```java
public class VoiatileTest {

    public static void main(String[] args) {
        VoiatileTestRunable myRunable = new VoiatileTestRunable();

        new Thread(myRunable,"t1").start();
        new Thread(myRunable,"t2").start();
        
    }
}
class VoiatileTestRunable implements Runnable{
        
    public volatile int count = 5;
    
    @Override
    public void run() {
        while(count>0){
            try {
                Thread.sleep(500);
                sale();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
        } 
    }
    public void sale() {
        
        if (count > 0) {
            --count;
            System.out.println(Thread.currentThread().getName() + ",出售第" + (5 - count) + "张票");
        }
        
    }
}
```
执行结果

```java
t1,出售第1张票
t2,出售第1张票
t1,出售第3张票
t2,出售第3张票
t1,出售第5张票
t2,出售第5张票
```

可以看到，执行结果并不是我们预期的结果，一共卖出去`6`张票，但是count变量没有累加到6，这是由于`volatitle`只保证了内存可见性，但却没有保证操作的原子性，`++`，`--`操作都不是原子性的。

那么如何保证操作原子性呢，我们可以使用Java中提供的原子类`AtomicInteger`，代码如下：

```java
package com.constantine.daily.concurrency.atomicobj;

import java.util.concurrent.atomic.AtomicInteger;

public class VoiatileTest {

    public static void main(String[] args) {
        VoiatileTestRunable myRunable = new VoiatileTestRunable();

        new Thread(myRunable,"t1").start();
        new Thread(myRunable,"t2").start();

    }
}
class VoiatileTestRunable implements Runnable{

    public static volatile AtomicInteger count = new AtomicInteger(5);

    @Override
    public void run() {

        int cnt;

        while((cnt =count.getAndDecrement())>0){
            try {
                Thread.sleep(500);
                System.out.println(Thread.currentThread().getName() + ",出售第" + ((5 - cnt)+1) + "张票");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
}
```

执行结果：

```java
t2,出售第2张票
t1,出售第1张票
t1,出售第4张票
t2,出售第3张票
t1,出售第5张票
```
可以看到，一共卖出去5张票，因为AtomicInteger 保证了操作原子性。还可以注意到，count变量并没有从1逐一累加到5，这是因为虽然`count.getAndDecrement()`是原子操作，但是`sale`方法中if语句里的代码块并不是原子操作。两个线程同时执行打印语句，就会出现这种情况，但是对卖票这件事情没有什么影响，不会出现卖超的情况。

通过上面代码我们了解到，`volatile`虽然提供了可见性保证，但是并没有保证操作的原子性。

那么在什么情况下可以使用volatile关键字呢？

1. 写入变量不依赖变量当前值时。如果依赖当前值，将是 `获取-计算-写入` 三步操作（上述代码中的--count，此操作依赖于count的当前值），这三步操作不是原子性的，而volatile不保证原子性
2. 读写变量值时没有加锁。因为加锁本身已经保证了内存可见性，这时候不需要再将变量声明为volatile了
