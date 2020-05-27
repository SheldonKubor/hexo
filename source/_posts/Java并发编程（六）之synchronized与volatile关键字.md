---
title: Java并发编程（六）之synchronized关键字原理
date: 2020-04-20 09:34:04
tags: [多线程,并发,Java]
---

之前我们说过在线程安全问题中如何使用`synchronized`关键字，现在我们看看，此关键字的原理是什么。

在多线程并发编程中`synchronized`一直是元老级的应用，虽说之前Java的版本中，这是一个重量级的锁，但是`Java SE 1.6`版本对此关键字进行了各种优化，有些情况下，它就表现的没那么笨重了。

先来看下利用synchronized实现同步的基础：Java中的每一个对象都可以作为锁。具体表现为以下3种形式。

1. 对于普通同步方法，锁是当前实例对象。
2. 对于静态同步方法，锁是当前类的Class对象。
3. 对于同步方法块，锁是Synchonized括号里配置的对象。

当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。若是线程在退出或者抛出异常的时候没有释放锁，那么共享资源将不会被释放，其他线程不能访问，程序就会陷入死锁。

在JVM中，同步的实现是通过监视器锁，也就是`Monitor`对象的进入和退出实现的。要么显示得通过 `monitorenter` 和 `monitorexit` 指令实现，要么隐示地通过方法调用和返回指令实现。

对于Java代码来说，或许最常用的同步实现就是同步方法。其中同步代码块是通过使用 `monitorenter` 和 `monitorexit` 实现的，而同步方法却是使用 `ACC_SYNCHRONIZED` 标记符隐示的实现，原理是通过方法调用指令检查该方法在常量池中是否包含 `ACC_SYNCHRONIZED` 标记符(这个我还没有去了解，所以就先略过，之后了解了再一起学习吧！)

`monitorenter`指令是在编译后插入到同步代码块的开始位置，而`monitorexit`是插入到方法结束处和异常处，JVM要保证每个`monitorenter`必须有对应的`monitorexit`与之配对。任何对象都有一个`monitor`与之关联，当且一个`monitor`被持有后，它将处于锁定状态。线程执行到`monitorenter`指令时，将会尝试获取对象所对应的`monitor`的所有权，即尝试获得对象的锁。线程执行到`monitorexit`指令时，将会释放对象所对应的`monitor`的所有权。

举个具体的例子看下

```java
public class SynchronizedDemo {
    public static synchronized void staticMethod() throws InterruptedException {
        System.out.println("静态同步方法开始");
        Thread.sleep(1000);
        System.out.println("静态同步方法结束");
    }
    public synchronized void method() throws InterruptedException {
        System.out.println("实例同步方法开始");
        Thread.sleep(1000);
        System.out.println("实例同步方法结束");
    }
    public synchronized void method2() throws InterruptedException {
        System.out.println("实例同步方法2开始");
        Thread.sleep(3000);
        System.out.println("实例同步方法2结束");
    }

    public void method3() throws InterruptedException {
        synchronized(this){
            System.out.println("同步代码块方法3开始");
            Thread.sleep(3000);
            System.out.println("同步代码块方法3结束");
        }
        
    }

    public static void main(String[] args) {
        final SynchronizedDemo synDemo = new SynchronizedDemo();
        Thread thread1 = new Thread(() -> {
            try {
               synDemo.method();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        Thread thread2 = new Thread(() -> {
            try {
                synDemo.method2();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        thread1.start();
        thread2.start();
    }
}
```
使用`javac SynchronizedDemo.java`编译一下，然后使用`javap -v SynchronizedDemo`反编译class文件，可以看到行号、本地变量表信息、反编译汇编代码，还会输出当前类用到的常量池等信息。

我们看一下反编译之后的同步方法与同步代码块

1. 同步方法
```java
public synchronized void method2() throws java.lang.InterruptedException;
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #11                 // String 实例同步方法2开始
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: ldc2_w        #12                 // long 3000l
        11: invokestatic  #7                  // Method java/lang/Thread.sleep:(J)V
        14: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        17: ldc           #14                 // String 实例同步方法2结束
        19: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        22: return
      LineNumberTable:
        line 13: 0
        line 14: 8
        line 15: 14
        line 16: 22
    Exceptions:
      throws java.lang.InterruptedException
```
可以看到，同步方法`flags`包含一个`ACC_SYNCHCRONIZED`标记符
区别于同步代码块的监视器实现，同步方法通过使用 `ACC_SYNCHRONIZED` 标记符隐示的实现
原理是通过方法调用指令检查该方法在常量池中是否包含 `ACC_SYNCHRONIZED` 标记符，如果有，JVM 要求线程在调用之前请求锁

2. 同步代码块
```java
public void method3() throws java.lang.InterruptedException;
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #15                 // String 同步代码块方法3开始
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: ldc2_w        #12                 // long 3000l
        15: invokestatic  #7                  // Method java/lang/Thread.sleep:(J)V
        18: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        21: ldc           #16                 // String 同步代码块方法3结束
        23: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        26: aload_1
        27: monitorexit
        28: goto          36
        31: astore_2
        32: aload_1
        33: monitorexit
        34: aload_2
        35: athrow
        36: return
      Exception table:
         from    to  target type
             4    28    31   any
            31    34    31   any
      LineNumberTable:
        line 19: 0
        line 20: 4
        line 21: 12
        line 22: 18
        line 23: 26
        line 25: 36
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 31
          locals = [ class SynchronizedDemo, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
    Exceptions:
      throws java.lang.InterruptedException
```
可以看到，在`flags`中没有了`ACC_SYNCHCRONIZED`标记符，并且在`Code`中包含了`monitorenter`和`monitorexit`，并且`monitorenter`在同步代码开始之前，`monitorexit`在同步代码结束之后。（可以看到，里面有两次`monitorexit`，这是因为第1次为同步正常退出释放锁；第2次为发生异步退出释放锁；这上面锁住的就是this。）

简单说明一下`monitorenter`和`monitorexit`
1. monitorenter：每个对象都是一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：
    - 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者；
    - 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1；（只有首先获得锁的线程才能允许继续获取多个锁）
    - 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权；

2. monitorexit：执行monitorexit指令的线程必须是对象实例所对应的监视器的所有者，指令执行时，线程会先将进入次数-1，若-1之后进入次数变成0，则线程退出监视器(即释放锁)，其他阻塞在该监视器的线程可以重新竞争该监视器的所有权

