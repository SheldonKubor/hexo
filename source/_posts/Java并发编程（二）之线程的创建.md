---
title: Java并发编程（二）之线程的创建
date: 2019-11-17 16:01:37
tags: [多线程,并发,Java]
---

上一篇讲了什么是并发，什么是多线程，这次我们讲一讲如何创建一个线程.

1. 继承Thread类，并重写其run方法
```java
public class ThreadTest {

    public static void main(String[] args) {
        //创建线程
        MyThread myThread = new MyThread();
        //启动线程
        myThread.start();
    }   
}
/**
 * 继承Thread类并重写其run方法
 */
class MyThread extends Thread{
    @Override
    public void run() {
        System.out.println("I am a Thread");
    }
} 
```

上述方式直接继承了Thread类，并重写了run方法， 在main方法中创建了一个线程实例，并调用其start方法启动了线程。需要注意的是，当创建完线程实例之后，线程并没有启动，直到调用了run方法后才真正的启动了线程。

其实调用了start方法后，线程也没有马上执行，而是处于了就绪状态（线程有多种状态，例如就绪，挂起，运行等，以后再详细说），就绪状态就是指线程已经获取了除了CPU资源外其他所有的所需资源，，一旦获取到CPU使用权限，就会真正处于运行状态，也就是运行run方法中的代码，直到run方法执行完毕，进程就变为终止状态。

使用继承Thread类这种方式的好处是，在run方法中获取当前线程，直接使用this关键字就可以了，不用使用`Thread.currentThread()`方法；缺点是Java是不支持多继承的，继承了Thread就不能继承别的类，另外就是线程任务没有与代码分离，多个线程执行同样的任务时，需要重复多分任务代码。

2. 实现Runable接口并重写run方法
```java
public class RunableTest {

    public static void main(String[] args) {
        //实例化任务类
        MyRunable myRunable = new MyRunable();
        //将任务实例作为参数创建线程并启动
        new Thread(myRunable).start();
        new Thread(myRunable).start();
    }
}
/**
 * 实现Runable接口
 */
class MyRunable implements Runnable {
    @Override
    public void run() {
        System.out.println("I am a Thread");
    }
}
```

上述方式，实现Runable接口，避免了占用继承的位置，同时，任务与代码分离，两个线程共用同一个task代码逻辑。但是无论是继承Thread类还是实现Runable接口来创建线程（这两种方式也是我们大学时候Java程序设计课上，老师提到的两种方法），都有一个缺点，那就是任务没有返回值，大家也可以看到run方法函数签名中的返回类型是`void`。下面看最后一种有返回值的方式-使用`FutureTask`

3. 使用FutureTask
```java
public class CallableTest {

    public static void main(String[] args) {
        //创建异步任务
        FutureTask<String> futureTask = new FutureTask<>(new MyCallable());
        //将任务作为参数新建线程并启动
        new Thread(futureTask).start();
        try {
            //获取执行结果
            String result = futureTask.get();
            System.out.println(result);
        } catch (Exception e) {
            e.printStackTrace();
        }
        
    }
}
/**
 * 创建任务类，类似于Runable
 */
class MyCallable implements Callable<String>{
    @Override
    public String call() throws Exception {
        System.out.println("I am a Thread");
        return "hello world";
    }
}
```

如上，代码中MyCallable类实现了Callable接口的call方法，在main中创建一个FutureTask对象，构造函数为Callable的实例，然后使用创建的FutureTask实例作为任务创建了一个线程并且启动，最后通过`futureTask.get()`等待任务执行完毕获取任务的返回结果。

除了这三种方法，其实还有一种高级用法，那就是使用线程池，这里就暂时先不详细的说了，简单说一说线程池主要解决的问题吧：
    
    1. 当执行大量异步任务时，每当需要执行异步任务就要new一个线程来运行，而线程的创建和销毁都是需要开销的。线程池中的线程可以复用，不需要每次执行任务都创建销毁。
    2. 线程池提供了一些对资源限制和管理的手段，比如限制线程个数，动态新增线程等，线程池也保留了一些基本的数据统计，比如当前线程完成的任务数目等。

线程池的使用，以及其原理，篇幅比较长，这篇文章就暂时不讲了，我们后续再讲，这篇就先到这吧。 