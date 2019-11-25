---
title: Java并发编程（三）之线程池的使用
date: 2019-11-24 16:40:14
tags: [多线程,并发,Java]
---

之前我们在如何创建线程的文章中介绍到了线程池创建线程，接下来学习一下如何使用线程池创建线程。

Java通过Executors提供四种常用线程池，这四种线程池都是通过`ThreadPoolExecutor`实现的，我们来看一下其构造方法

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

构造方法内具体的逻辑我们先忽略，先来看看构造方法的参数含义：

1. corePoolSize: 线程池核心线程个数
2. maximumPoolSize： 线程池最大线程个数
3. keepAliveTime： 线程存活时间，指的是如果当前线程池中的线程数量比核心线程数量多，并且是闲置状态，则这些闲置线程能存活的最大时间
4. unit： 存活时间的时间单位
5. workQueue： 用于保存等待执行的任务的阻塞队列，比如基于数组的有界队列`ArrayBlockingQueue`，基于链表的`LinkedBlockingQueue`等
6. threadFactory： 创建线程的工厂
7. handler： 拒绝策略，当队列满了，并且线程个数达到maximumPoolSize后采取的拒绝策略

`Executors`中创建线程池的方法其实就是通过使用`ThreadPoolExecutor`构造方法的不同参数来创建不同类型的线程池 

接下来我们深入源码看一看`Executors`中创建线程池的方法，它是如何使用`ThreadPoolExecutor`来创建线程池的

1. 单线程化线程池（SingleThreadExecutor）

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

- 特点：创建一个核心线程个数和最大线程个数都为1的线程池，并且阻塞队列长度为Integer.MAX_VALUE。keepAliveTime为0说明只要线程个数比核心线程数多并且当前空闲则回收。
- 应用场景：不适合并发但可能引起IO阻塞性及影响UI线程响应的操作，如数据库操作、文件操作等。

使用示例：

```java
ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();

for (int i = 0; i < 10; i++) {
    final int index = i;
    singleThreadExecutor.execute(new Runnable() {

        @Override
        public void run() {
            try {
                System.out.println(index);
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
}
```

执行结果为

```java
0
1
2
3
4
5
6
7
8
9
```

可以看到，index的值是递增的，因为只有一个线程在执行任务

2. 定长线程池（FixedThreadPool）

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}

public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory);
}
```

- 特点：只有核心线程，线程数量固定（其`corePoolSize`和`maximumPoolSize`值相同），执行完立即回收（`keepAliveTime`为0），阻塞队列长度为Integer.MAX_VALUE。
- 应用场景：控制线程最大并发数。

使用示例：

```java
ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);
for (int i = 0; i < 10; i++) {
    final int index = i;

    fixedThreadPool.execute(new Runnable() {
        @Override
        public void run() {
            try {
                System.out.println(index);
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
}
```

执行结果为：

```java
0
2
1
3
4
5
6
7
8
9
```

可以看到，index的值不是递增的，因为只有三个线程在执行任务

3. 可缓存线程池（CachedThreadPool）

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}

public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>(),
                                    threadFactory);
}
```

- 特点：无核心线程，非核心线程数量无限，执行完闲置60s后回收，这个类型的特殊之处在于，加入同步队列的任务会被马上执行，同步队列里面最多只有一个任务（因为每来一个任务就会马上创建一个线程来执行）
- 应用场景：执行大量、耗时少的任务。

使用示例：

```java
ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
for (int i = 0; i < 10; i++) {
    final int index = i;
    cachedThreadPool.execute(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(2000);
                System.out.println(index);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
        }
    });
}
```

执行结果：

```java
8
6
2
4
0
3
1
5
7
9
```

可以看到，index的值不是递增的，原因与上一个相同。

4. 定时线程池（ScheduledThreadPool）

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```
- 注：这个与上面三个有点区别，这是使用的是`ScheduledThreadPoolExecutor`创建的，此类继承`ThreadPoolExecutor`类。

- 特点：核心线程数量固定，非核心线程数量无限，执行完闲置10ms后回收，任务队列为延时阻塞队列。
- 应用场景：执行定时或周期性的任务。

使用示例：

```java
ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
scheduledThreadPool.schedule(new Runnable() {
    @Override
    public void run() {
        System.out.println("delay 3 seconds");
    }
}, 3, TimeUnit.SECONDS);
```

执行结果：
    延迟3秒后打印出`delay 3 seconds`

下面我们来用代码具体看下，这些线程池的区别

1. 单线程化线程池（SingleThreadExecutor）

```java
// 创建一个单线程的线程池  
ExecutorService pool = Executors.newSingleThreadExecutor();  
// 创建线程  
Thread t1 = new MyThreadPoolTest();  
Thread t2 = new MyThreadPoolTest();  
Thread t3 = new MyThreadPoolTest();  
Thread t4 = new MyThreadPoolTest();  
Thread t5 = new MyThreadPoolTest();  
// 将线程放入池中进行执行  
pool.execute(t1);  
pool.execute(t2);  
pool.execute(t3);  
pool.execute(t4);  
pool.execute(t5);  
// 关闭线程池  
pool.shutdown();
```
执行结果：

```java
pool-1-thread-1正在执行。。。
pool-1-thread-1正在执行。。。
pool-1-thread-1正在执行。。。
pool-1-thread-1正在执行。。。
pool-1-thread-1正在执行。。。
```

通过执行结果我们可以看到，虽然我们在线程池中放了5个线程，但是由于线程池是单线程池，所以只有第一个线程存在于线程池中。
每次调用execute方法，其实最后都是调用了thread-1的run方法。

2. 定长线程池（FixedThreadPool）

```java
ExecutorService pool = Executors.newFixedThreadPool(5);
// 创建线程  
Thread t1 = new MyThreadPoolTest();  
Thread t2 = new MyThreadPoolTest();  
Thread t3 = new MyThreadPoolTest();  
Thread t4 = new MyThreadPoolTest();  
Thread t5 = new MyThreadPoolTest();  
// 将线程放入池中进行执行  
pool.execute(t1);  
pool.execute(t2);  
pool.execute(t3);  
pool.execute(t4);  
pool.execute(t5);  
// 关闭线程池  
pool.shutdown();
```

```java
pool-1-thread-1正在执行。。。
pool-1-thread-4正在执行。。。
pool-1-thread-3正在执行。。。
pool-1-thread-2正在执行。。。
pool-1-thread-5正在执行。。。
```

将`Executors.newFixedThreadPool(5)`中的`5`改为`2`再试试

```java
pool-1-thread-1正在执行。。。
pool-1-thread-2正在执行。。。
pool-1-thread-2正在执行。。。
pool-1-thread-1正在执行。。。
pool-1-thread-2正在执行。。。
```

从以上结果可以看出，newFixedThreadPool的参数指定了可以运行的线程的最大数目，超过这个数目的线程加进去以后，不会运行。其次，加入线程池的线程属于托管状态，线程的运行不受加入顺序的影响。


3. 可缓存线程池（CachedThreadPool）

```java
ExecutorService pool = Executors.newCachedThreadPool();
// 创建线程  
Thread t1 = new MyThreadPoolTest();  
Thread t2 = new MyThreadPoolTest();  
Thread t3 = new MyThreadPoolTest();  
Thread t4 = new MyThreadPoolTest();  
Thread t5 = new MyThreadPoolTest();  
// 将线程放入池中进行执行  
pool.execute(t1);  
pool.execute(t2);  
pool.execute(t3);  
pool.execute(t4);  
pool.execute(t5);  
// 关闭线程池  
pool.shutdown();
```

```java
pool-1-thread-1正在执行。。。
pool-1-thread-3正在执行。。。
pool-1-thread-2正在执行。。。
pool-1-thread-5正在执行。。。
pool-1-thread-4正在执行。。。
```

这种方式的特点是：可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们。

4. 定时线程池（ScheduledThreadPool）

```java
ScheduledThreadPoolExecutor exec = new ScheduledThreadPoolExecutor(1);
exec.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        System.out.println("================");
    }
}, 1000, 5000, TimeUnit.MILLISECONDS);

exec.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        System.out.println(System.nanoTime());
    }
}, 1000, 2000, TimeUnit.MILLISECONDS);
```

```java
================
233471023732097
233473026158692
233475023768285
================
233477030024240
233479027432148
================
233481026928538
233483025530535
233485025312657
================
```

可以看到，两个任务互不影响

上面的代码建议大家亲自执行以下，会有更深的体会。
