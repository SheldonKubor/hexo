---
title: Java并发编程（九）之指令重排序
date: 2020-04-28 09:30:37
tags: [多线程,并发,Java]
---

Java内存模型允许编译器和处理器对指令进行重排序以提高程序性能，但是只会对没有数据依赖性的指令进行重排序。单线程的情况下，重排序可以保证运行结果与不进行重排序运行结果一致，但在多线程下就会存在问题。

举个例子：

```java
int a = 1;      //(1)
int b = 2;      //(2)
int c = a + b;  //(3)
```

在上述代码中，变量 c 的值依赖于 a 和 b ，所以在重排序之后，需要保证(1)和(2)的操作在(3)之前，c的值就不会受到影响，但是(1)和(2)的操作谁先执行在重排序之后并不能确定。这样的情况在单线程下不会出现问题，因为单线程下不会影响最终结果。

下面看个多线程的例子

```java
public class InstructionReorder {

    private static int num = 0;
    private static boolean ready = false;

    public static class ReadThread extends Thread {
        @Override
        public void run() {
            while (!Thread.currentThread().isInterrupted()){
                if(ready){ //(1)
                    System.out.println(num+num); //(2)
                }
                System.out.println("read thread...");
            }
        }
    }

    public static class WriteThread extends Thread {
        @Override
        public void run() {
            num = 2; //(3)
            ready = true; //(4)
            System.out.println("write thread set over");
        }
    }

    public static void main(String[] args) throws InterruptedException{
        ReadThread readThread = new ReadThread();
        readThread.start();

        WriteThread writeThread = new WriteThread();
        writeThread.start();

        Thread.sleep(10);
        readThread.interrupt();
        System.out.println("main exit");
    }

}
```

这段代码里变量都没有被`volatile`修饰，也没有任何同步措施，在多线程的情况下存在内存可见性问题。但先不考虑这个问题，在不考虑内存可见性问题的情况下，这段程序一定会输出4吗？答案是不一定，由于代码(1)(2)和(3)(4)之间不存在依赖关系，所以(3)(4)可能被重排序为先执行(4)，再执行(3)，那么执行(4)之后，读线程可能已经执行了(1)操作，并且在(3)操作执行前开始执行(2)操作，这时候输出的结果是0而不是4。

解决方法就是使用`volatile`修饰共享变量，可以避免重排序与内存可见性问题。

写`volatile`变量时，可以确保`volatile`写之前的操作不会被重排序到`volatile`写之后，读`volatile`变量时，可以确保`volatile`读之后的操作不会被编译器重排序到`volatile`读之前。