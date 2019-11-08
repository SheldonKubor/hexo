---
title: Redis中的数据结构之简单动态字符串（二）
date: 2019-11-08 20:10:42
tags: [Redis,数据结构]
---

上一篇我们讲到了Redis中简单动态字符串与c语言中字符串相比的一个优势，也就是获取字符串长度时的时间复杂度有了大幅度降低，满足了Redis在效率方面的要求。

今天来说一下，SDS是如何满足Redis对字符串在安全性方面的要求。

除了获取字符串长度的复杂度高之外，C语言中字符串不记录自身长度带来的另一个问题是容易造成缓冲区溢出（buffer overflow），例如`<string.h>/strcat`函数可以将`src`字符串中的内容拼接到`dest`字符串末尾。

```c++
char* strcat(char* dest,const char* src)
```

举个例子，假如一段程序，在内存中有两个紧紧相邻的c语言字符串`s1`和`s2`与`Redis`,`MongoDB`，如图所示

![在内存中紧邻的两个C字符串](/pic_doc/c_string_s1&s2.jpeg) 

如果一个程序员决定通过使用`strcat`函数来修改`s1`的内容为`Redis Cluster`

```c++
strcat(s1, "Cluster");
```

但他忘了在执行`strcat`之前为`s1`分配足够的空间（众所周知，在c语言与c++中需要程序员自己手工对内存进行管理），那么就会导致`Cluster`溢出到`s2`的位置，如下图：

![s2被溢出的内容覆盖](/pic_doc/c_string_s1&s2_buf_overflow.jpeg) 

而在SDS中，通过一个巧妙的方法避免了这种情况的发生，也就是利用SDS结构体中的`free`属性(在最新版的Redis中，SDS的结构体中已经移除了free属性，转而使用alloc来代替，不过两个属性的含义有所不同，但在SDS中起到的作用基本一致，这个有机会再说哈)。那么SDS是怎么使用`free`属性来杜绝缓冲区溢出的呢？

举个例子：SDS的API中也有一个用于执行拼接操作的`sdscat`函数，类似于c语言中的`strcat`，当SDS中`sdscat`需要拼接字符串的时候，会先检查SDS的空间是否满足所需的要求，如果不满足的话，Redis会自动将SDS的空间扩展到所需大小，然后再执行拼接操作，所以使用SDS既不必像c语言中手动修改空间大小，也不会出现前面所说的c语言中字符串的缓冲区问题。（`注：这里只是举了一个拼接字符串的例子，其他的有关SDS的API在修改SDS的时候也会做上面的操作`）

例如有以下SDS：

![执行sdscat之前](/pic_doc/redis_sds.jpeg)

我们执行：

```c++
sdscat(s,"Cluster");
```

拼接完的SDS如下图所示：

![执行sdscat之后](/pic_doc/sds_after_sdscat.jpeg)

注意：上图所示的SDS，sdscat不仅对SDS进行了拼接操作，还为SDS分配了13字节的未使用空间，并且拼接之后的字符串正好也是13字节长，这种现象既不是巧合，也不是bug(工业界以及大规模使用的Redis怎么会有bug！？)，这和SDS的空间分配策略有关系，空间分配策略又是Redis一个很巧妙的设计，所以，我们下一篇再讲吧！


