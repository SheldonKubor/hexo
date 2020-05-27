---
title: Java如何快速读取大文件
date: 2020-05-20 10:16:47
tags: [大文件,NIO,Java]
---

最近看到一种快速读取大文件的方法，记录一下，与平时常用的方法做一下对比。

1. 普通输入流

```java
/**
    * 普通输入流
    * @param filename
    */
public static void inputStream(Path filename) {
    try (InputStream is = Files.newInputStream(filename)) {
        int c;
        while((c = is.read()) != -1) {

        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

2. 带缓冲的输入流

```java
/**
    * 带缓冲的输入流
    * @param filename
    */
public static void bufferedInputStream(Path filename) {
    try (InputStream is = new BufferedInputStream(Files.newInputStream(filename))) {
        int c;
        while((c = is.read()) != -1) {

        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

3. 随机访问文件

```java
/**
    * 随机访问文件
    * @param filename
    */
public static void randomAccessFile(Path filename) {
    try (RandomAccessFile randomAccessFile  = new RandomAccessFile(filename.toFile(), "r")) {
        for (long i = 0; i < randomAccessFile.length(); i++) {
            randomAccessFile.seek(i);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

4. 内存映射文件

```java
/**
    * 内存映射文件
    * @param filename
    */
public static void mappedFile(Path filename) {
    try (FileChannel fileChannel = FileChannel.open(filename)) {
        long size = fileChannel.size();
        MappedByteBuffer mappedByteBuffer = fileChannel.map(FileChannel.MapMode.READ_ONLY, 0, size);
        for (int i = 0; i < size; i++) {
            mappedByteBuffer.get(i);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

5. 测试主类 

```java
public class IOTest {
    public static void main(String[] args) {
        //测试文件大小为987.2M
        long start = System.currentTimeMillis();
        //bufferedInputStream(Paths.get("/Users/constantine/Downloads/jisuzhuisha.mp4"));//22038
        //inputStream(Paths.get("/Users/constantine/Downloads/jisuzhuisha.mp4")); //龟速，没有耐心等出结果
        //randomAccessFile(Paths.get("/Users/constantine/Downloads/jisuzhuisha.mp4"));//龟速，没有耐心等出结果
        mappedFile(Paths.get("/Users/constantine/Downloads/jisuzhuisha.mp4"));//816
        long end = System.currentTimeMillis();
        System.out.println(end-start);

    }   
}

```

可以看到，内存映射文件的方式比传统方式快了N倍，这就是Java NIO的威力。

- 简单介绍一下内存映射文件：

所谓内存映射文件，就是将文件映射到内存，文件对应于内存中的一个字节数组，对文件的操作变为对这个字节数组的操作，而字节数组的操作直接映射到文件上。这种映射可以是映射文件全部区域，也可以是只映射一部分区域。

不过，这种映射是操作系统提供的一种假象，文件一般不会马上加载到内存，操作系统只是记录下了这回事，当实际发生读写时，才会按需加载。操作系统一般是按页加载的，页可以理解为就是一块，页的大小与操作系统和硬件相关，典型的配置可能是4K, 8K等，当操作系统发现读写区域不在内存时，就会加载该区域对应的一个页到内存。

这种按需加载的方式，使得内存映射文件可以方便处理非常大的文件，内存放不下整个文件也不要紧，操作系统会自动进行处理，将需要的内容读到内存，将修改的内容保存到硬盘，将不再使用的内存释放。

在应用程序写的时候，它写的是内存中的字节数组，这个内容什么时候同步到文件上呢？这个时机是不确定的，由操作系统决定，不过，只要操作系统不崩溃，操作系统会保证同步到文件上，即使映射这个文件的应用程序已经退出了。

在一般的文件读写中，会有两次数据拷贝，一次是从硬盘拷贝到操作系统内核，另一次是从操作系统内核拷贝到用户态的应用程序。而在内存映射文件中，一般情况下，只有一次拷贝，且内存分配在操作系统内核，应用程序访问的就是操作系统的内核内存空间，这显然要比普通的读写效率更高。

内存映射文件的另一个重要特点是，它可以被多个不同的应用程序共享，多个程序可以映射同一个文件，映射到同一块内存区域，一个程序对内存的修改，可以让其他程序也看到，这使得它特别适合用于不同应用程序之间的通信。

操作系统自身在加载可执行文件的时候，一般都利用了内存映射文件，比如：

1. 按需加载代码，只有当前运行的代码在内存，其他暂时用不到的代码还在硬盘
2. 同时启动多次同一个可执行文件，文件代码在内存也只有一份
3. 不同应用程序共享的动态链接库代码在内存也只有一份 
4. 内存映射文件也有局限性，比如，它不太适合处理小文件，它是按页分配内存的，对于小文件，会浪费空间，另外，映射文件要消耗一定的操作系统资源，初始化比较慢。

简单总结下，对于一般的文件读写不需要使用内存映射文件，但如果处理的是大文件，要求极高的读写效率，比如数据库系统，或者需要在不同程序间进行共享和通信，那就可以考虑内存映射文件。