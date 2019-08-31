---
title: Java中的instanceof关键字
date: 2019-08-27 15:56:57
tags: [Java基础]
---


instanceof是Java的一个二元操作符，和==，>，<是同一类。由于它是由字母组成的，所以也是Java的保留关键字。它的作用是测试它左边的对象是否是它右边的类的实例，返回boolean类型的数据（true，false）

接下来让我们实际体验一下此关键字的作用

首先我们定义一个类Obj1（为了简单，我们就不在类中定义任何属性与方法了）
```
class Obj1{

}
```
然后来测试一下instanceof

```
Obj1 obj1 = new Obj1()
System.out.println(obj1 instanceof Obj1);
```
输出结果为 `true`

若类存在继承关系呢？，我们来试一下

首先，定义Obj2，并使其继承Obj1

```
class Obj2 extends Obj1{

}
```
然后，测试一下instanceof
```
Obj2 obj2 = new Obj2();
System.out.println(obj2 instanceof Obj1);

```

输出结果为 `true`
可以看到，由于Obj2继承了Obj1，所以obj2也是Obj1的实例

那反过来呢？

```
Obj1 obj1 = new Obj1();
System.out.println(obj1 instanceof Obj2);
```
可以看到输出结果为 `false`


若是某一个类有接口的继承关系，与上面类的继承关系也一样，大家可以自己试一试。
