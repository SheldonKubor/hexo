---
title: Java代理模式
date: 2020-05-15 10:17:17
tags: [设计模式,jdk动态代理,cglib]
---

代理模式的含义是，给某一个对象提供一个代理，并由代理对象来控制对真实对象的访问。代理模式是一种结构型设计模式。

代理模式角色分为 3 种：

Subject（抽象主题角色）：定义代理类和真实主题的公共对外方法，也是代理类代理真实主题的方法；

RealSubject（真实主题角色）：真正实现业务逻辑的类；

Proxy（代理主题角色）：用来代理和封装真实主题；

先来看看静态代理

编写一个接口（抽象主题角色）

```java
public interface UserService {
    public void select();   
    public void update();
}
```

然后编写该接口的实现类（真实主题角色）

```java
public class UserServiceImpl implements UserService {  
    public void select() {  
        System.out.println("查询 selectById");
    }
    public void update() {
        System.out.println("更新 update");
    }
}
```

写一个代理类 UserServiceProxy（代理主题角色），代理类需要实现`UserService`，对`UserServiceImpl`进行功能一些增强

```java
public class UserServiceProxy implements UserService {
    private UserService target; // 被代理的对象

    public UserServiceProxy(UserService target) {
        this.target = target;
    }
    public void select() {
        before();
        target.select();    // 这里才实际调用真实主题角色的方法
        after();
    }
    public void update() {
        before();
        target.update();    // 这里才实际调用真实主题角色的方法
        after();
    }

    private void before() {     // 在执行方法之前执行
        System.out.println(String.format("log start time [%s] ", new Date()));
    }
    private void after() {      // 在执行方法之后执行
        System.out.println(String.format("log end time [%s] ", new Date()));
    }
}
```

然后测试一下

```java
public class Client1 {
    public static void main(String[] args) {
        UserService userServiceImpl = new UserServiceImpl();
        UserService proxy = new UserServiceProxy(userServiceImpl);//传入需要被代理的角色

        proxy.select();
        proxy.update();
    }
}
```

输出：

```java
log start time [Fri May 15 10:27:28 CST 2020] 
查询 selectById
log end time [Fri May 15 10:27:28 CST 2020] 
log start time [Fri May 15 10:27:28 CST 2020] 
更新 update
log end time [Fri May 15 10:27:28 CST 2020] 
```

可以看到，我们通过代理类对`userServiceImpl`的两个方法进行了增强，在方法执行前后打印了一些日志。

大家可能会想到，Spring中AOP功能，也可以在一些方法执行前后打印一些日志，做一些操作。其实这也是用了代理模式，但是SpringAOP中用的并不是上述的简单的静态代理，而是使用的动态代理，jdk动态代理，或者cglib动态代理。

静态代理有一些优缺点我们来看一下：

- 静态代理优点
1. 达到了功能增强的目的，而且没有侵入原代码

- 静态代理缺点
1. 当需要代理多个类的时候，由于代理对象要实现与目标对象一致的接口，有两种方式：
   只维护一个代理类，由这个代理类实现多个接口，但是这样就导致代理类过于庞大 
   新建多个代理类，每个目标对象对应一个代理类，但是这样会产生过多的代理类
2. 当接口需要增加、删除、修改方法的时候，目标对象与代理类都要同时修改，不易维护。

一句话总结缺点，不够灵活简便。

那么来看看动态代理

1. jdk动态代理

jdk动态代理主要涉及两个类：`java.lang.reflect.Proxy` 和 `java.lang.reflect.InvocationHandler`，一看到`reflect`包，应该就能想到动态代理利用看Java的反射特性。

我们还编写一个代理类，实现上述静态代理中打印日志的功能

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Date;

public class LogHandler implements InvocationHandler {
    Object target;  // 被代理的对象，实际的方法执行者

    public LogHandler(Object target) {
        this.target = target;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);  // 调用 target 的 method 方法
        after();
        return result;  // 返回方法的执行结果
    }
    // 调用invoke方法之前执行
    private void before() {
        System.out.println(String.format("log start time [%s] ", new Date()));
    }
    // 调用invoke方法之后执行
    private void after() {
        System.out.println(String.format("log end time [%s] ", new Date()));
    }
}
```

写一个主类来测试
```java
public class Client2 {
    public static void main(String[] args) throws IllegalAccessException, InstantiationException {
        // 1. 创建被代理的对象，UserService接口的实现类
        UserServiceImpl userServiceImpl = new UserServiceImpl();
        // 2. 获取对应的 ClassLoader
        ClassLoader classLoader = userServiceImpl.getClass().getClassLoader();
        // 3. 获取所有接口的Class，这里的UserServiceImpl只实现了一个接口UserService，
        Class[] interfaces = userServiceImpl.getClass().getInterfaces();
        // 4. 创建一个将传给代理类的调用请求处理器，处理所有的代理对象上的方法调用
        //     这里创建的是一个自定义的日志处理器，须传入实际的执行对象 userServiceImpl
        InvocationHandler logHandler = new LogHandler(userServiceImpl);
        /*
		   5.根据上面提供的信息，创建代理对象 在这个过程中，
               a.JDK会通过根据传入的参数信息动态地在内存中创建和.class 文件等同的字节码
               b.然后根据相应的字节码转换成对应的class，
               c.然后调用newInstance()创建代理实例
		 */
        UserService proxy = (UserService) Proxy.newProxyInstance(classLoader, interfaces, logHandler);
        // 调用代理的方法
        proxy.select();
        proxy.update();
    }
}
```

执行结果
```java
log start time [Fri May 15 10:38:31 CST 2020] 
查询 selectById
log end time [Fri May 15 10:38:31 CST 2020] 
log start time [Fri May 15 10:38:31 CST 2020] 
更新 update
log end time [Fri May 15 10:38:31 CST 2020] 
```

InvocationHandler 和 Proxy 的主要方法介绍如下：

 - java.lang.reflect.InvocationHandler
`Object invoke(Object proxy, Method method, Object[] args)` 定义了代理对象调用方法时希望执行的动作，用于集中处理在动态代理类对象上的方法调用

- java.lang.reflect.Proxy
`static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)` 构造实现指定接口的代理类的一个新实例，所有方法会调用给定处理器对象的 invoke 方法

2. cglib动态代理

maven引入CGLIB包

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.3.0</version>
</dependency>
```

- 注意，从`Spring 3.2`开始， 不需要定义cglib依赖关系它已经被重新打包并直接集成在 `spring-core` 这个jar包中。

编写一个UserDao类，它没有接口，只有两个方法，select() 和 update()

```java
public class UserDao {
    public void select() {
        System.out.println("UserDao 查询 selectById");
    }
    public void update() {
        System.out.println("UserDao 更新 update");
    }
}
```

编写一个 LogInterceptor ，继承了 MethodInterceptor，用于方法的拦截回调

```java
public class LogInterceptor implements MethodInterceptor {
    /**
     * @param object 表示要进行增强的对象
     * @param method 表示拦截的方法
     * @param objects 数组表示参数列表，基本数据类型需要传入其包装类型，如int-->Integer、long-Long、double-->Double
     * @param methodProxy 表示对方法的代理，invokeSuper方法表示对被代理对象方法的调用
     * @return 执行结果
     * @throws Throwable
     */
    @Override
    public Object intercept(Object object, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        Object result = methodProxy.invokeSuper(object, objects);   // 注意这里是调用 invokeSuper 而不是 invoke，否则死循环，methodProxy.invokesuper执行的是原始类的方法，method.invoke执行的是子类的方法
        after();
        return result;
    }
    private void before() {
        System.out.println(String.format("log start time [%s] ", new Date()));
    }
    private void after() {
        System.out.println(String.format("log end time [%s] ", new Date()));
    }
}
```

测试类：

```java
public class CglibTest {
    public static void main(String[] args) {
        LogInterceptor logInterceptor = new LogInterceptor();
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(UserDao.class);   // 设置超类，cglib是通过继承来实现的
        enhancer.setCallback(logInterceptor);

        UserDao proxy = (UserDao) enhancer.create();   // 创建代理类
        proxy.select();
        proxy.update();

    }
}
```

执行结果：

```java
log start time [Fri May 15 10:50:36 CST 2020] 
UserDao 查询 selectById
log end time [Fri May 15 10:50:36 CST 2020] 
log start time [Fri May 15 10:50:36 CST 2020] 
UserDao 更新 update
log end time [Fri May 15 10:50:36 CST 2020] 
```

还可以进一步多个 MethodInterceptor 进行过滤筛选

定义`LogInterceptor2`

```java
public class LogInterceptor2 implements MethodInterceptor {
    @Override
    public Object intercept(Object object, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before();
        Object result = methodProxy.invokeSuper(object, objects);
        after();
        return result;
    }
    private void before() {
        System.out.println(String.format("log2 start time [%s] ", new Date()));
    }
    private void after() {
        System.out.println(String.format("log2 end time [%s] ", new Date()));
    }
}
```

定义一个回调过滤器，在cglib回调时可以设置对不同方法执行不同的回调逻辑，或者根本不执行回调。

```java
public class DaoFilter implements CallbackFilter {
    @Override
    public int accept(Method method) {
        if ("select".equals(method.getName())) {
            return 0;   // Callback 列表第1个拦截器 UserDao.select方法执行第一个拦截器
        }
        return 1;   // Callback 列表第2个拦截器，return 2 则为第3个，以此类推
    }
}
```

测试类：

```java
public class CglibTest2 {
    public static void main(String[] args) {
        LogInterceptor logInterceptor = new LogInterceptor();
        LogInterceptor2 logInterceptor2 = new LogInterceptor2();
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(UserDao.class);   // 设置超类，cglib是通过继承来实现的
        enhancer.setCallbacks(new Callback[]{logInterceptor, logInterceptor2, NoOp.INSTANCE});   // 设置多个拦截器，NoOp.INSTANCE是一个空拦截器，不做任何处理
        enhancer.setCallbackFilter(new DaoFilter());

        UserDao proxy = (UserDao) enhancer.create();   // 创建代理类
        proxy.select();
        proxy.update();
    }
}
```

运行结果

```java
log start time [Fri May 15 10:54:18 CST 2020] 
UserDao 查询 selectById
log end time [Fri May 15 10:54:18 CST 2020] 
log2 start time [Fri May 15 10:54:18 CST 2020] 
UserDao 更新 update
log2 end time [Fri May 15 10:54:18 CST 2020] 
```
可以看到
- select方法执行第一个拦截器，打印`log`
- update方法执行第二个拦截器，打印`log2`

cglib 创建动态代理类的模式是：

1. 查找目标类上的所有非final 的public类型的方法定义；
2. 将这些方法的定义转换成字节码；
3. 将组成的字节码转换成相应的代理的class对象；
4. 实现 MethodInterceptor接口，用来处理对代理类上所有方法的请求

jdk动态代理与cglib动态代理对比:

- jdk动态代理：基于Java反射机制实现，必须要实现了接口的业务类才能用这种办法生成代理对象。
- cglib动态代理：基于ASM机制实现，通过生成业务类的子类作为代理类。

jdk动态代理的优势：

1. 最小化依赖关系，减少依赖意味着简化开发和维护，JDK 本身的支持，可能比 cglib 更加可靠。
2. 平滑进行 JDK 版本升级，而字节码类库通常需要进行更新以保证在新版 Java 上能够使用。
3. 代码实现简单。

cglib动态代理的优势：

1. 无需实现接口，达到代理类无侵入
2. 只操作我们关心的类，而不必为其他相关类增加工作量。
3. 高性能

另外：

- Spring5 之前的版本 AOP 在默认情况下是使用 jdk 动态代理的，之后的版本默认使用 jdk 动态代理，如果对象没有实现接口，则使用 cglib 代理。
- 在 SpringBoot 1.5.x 版本中，默认还是使用 jdk 动态代理的。
- SpringBoot 2.x 为何默认使用 cglib


SpringBoot 2.x 版本为什么要默认使用 Cglib 来实现 AOP 呢？这么做的好处又是什么呢？

假设，我们有一个UserServiceImpl和UserService类，此时需要在UserContoller中使用UserService。在 Spring 中通常都习惯这样写代码：

```java
@Autowired
UserService userService;
```

在这种情况下，无论是使用 jdk 动态代理，还是 cglib 都不会出现问题。

但是，如果你的代码是这样的呢：

```java
@Autowired
UserServiceImpl userService;
```

这个时候，如果我们是使用 JDK 动态代理，那在启动时就会报错：

因为 JDK 动态代理是基于接口的，代理生成的对象只能赋值给接口变量。
而 CGLIB 就不存在这个问题。因为 CGLIB 是通过生成子类来实现的，代理对象无论是赋值给接口还是实现类这两者都是代理对象的父类。
SpringBoot 正是出于这种考虑，于是在 2.x 版本中，将 AOP 默认实现改为了 CGLIB。
