---
title: AOP和Spring AOP
abbrlink: 9732
date: 2022-11-16 23:33:41
description: 《Spring揭秘》读书笔记
---

# AOP

AOP 是一种理念，要实现这种理念通常需要一种现实的方式。与 OOP 需要相应的语言支持一样，AOP 也需要某种语言以帮助实现相应的概念实体，这些实现 AOP 的语言被统称为 AOL，即 Aspect-Oriented Language。

AOL 可以与系统实现语言相同，比如，如果系统实现语言为 Java，那么，相应的 AOL 也可以为 Java。但 AOL 并非一定要与系统实现语言相同，它也可以是其他语言，比如 AspectJ 是扩展自 Java 的一种 AOL，显然与 Java 的系统实现语言（C++）属于不同的两种语言。

Java 平台上的 AOP 实现机制：动态代理、动态字节码增强、Java代码生成、自定义类加载器、AOL 扩展（AspectJ）。

## AOP 的概念实体

### Joinpoint

在系统运行之前，AOP 的功能模块都需要织入到 OOP 的功能模块中。所以，要进行这种织入过程我们需要知道在系统的哪些执行点上进行织入操作，这些将要在其之上进行织入操作的系统执行点就称之为Joinpoint。

**常见的 Joinpoint 类型：**

- 方法调用（method call）：方法被调用时所处的程序执行点
- 方法执行（method execution）：方法内部执行开始时点

方法调用是在调用对象上的执行点，而方法执行则是在被调用到的方法逻辑执行的时点。对于同一对象，方法调用要先于方法执行。

- 构造方法调用：程序执行过程中某个对象调用其构造方法进行初始化的时点
- 构造方法执行：某个对象构造方法内部执行的开始时点
- 字段设置：对象的某个属性通过 setter 方法被设置 **或者** 直接被设置的时点
- 字段获取：某个对象相应属性被访问的时点，可以是 getter 方法访问 **或者** 是直接访问
- 异常处理执行：在某些异常类型抛出后，对应的异常处理逻辑执行的时点
- 类初始化：类中某些静态类型或者静态块的初始化时点

### Pointcut

Pointcut 概念代表的是 Joinpoint 的表述方式。将横切逻辑织入当前系统的过程中，需要参照 Pointcut 规定的 Joinpoint 信息，才可以知道应该往系统的哪些 Joinpoint 上织入横切逻辑。

> 我理解的 Pointcut 概念就是一组 Joinpoint 的集合。

**Pointcut 的表述方式：**

Pointcut 可以直接指定 Joinpoint 所在方法名称，也可以使用正则表达式，或者特定的 Pointcut 表述语言。

通常，Pointcut 与 Pointcut 之间还可以进行逻辑运算（&& ||）。这样，我们就可以从简单 Pointcut 开始，然后通过逻辑运算，得到最终需要的可能较为复杂的 Pointcut。

### Advice

Advice 是单一横切关注点逻辑的载体，它代表将会植入到 Joinpoint 的横切逻辑。如果将 Aspect 比作 OOP 中的 Class，那么 Advice 就相当于 Class 中的 Method。

按照 Advice 在 Joinpoint 位置执行实际的差异或者完成功能的不同，Advice 可以分成多种具体形式，如下：

1. **Before Advice：**在 Joinpoint 指定位置之前执行的 Advice 类型。
2. **After Advice：**在相应连接点之后执行的 Advice 类型，但该类型的 Advice 还可以细分为以下三种：
   1. After returning Advice：当前 Joinpoint 处执行流程正常完成后执行；
   2. After throwing Advice：在当前 Joinpoint 执行过程中抛出异常的情况下才执行；
   3. After Advice：不管 Joinpoint 处执行流程正常结束还是抛出异常都会执行，类似 finally 块。
3. **Around Advice：**对附加其上的 Joinpoint 进行 “包裹” 可以在 Joinpoint 之前和之后都指定相应的逻辑，甚至于中断或者忽略 Joinpoint 处原来程序流程的执行，比如 Servlet 的 Filter 功能就是一种 Around Advice 的体现。
4. **Introduction：**Introduction 可以为原有的对象添加新的特性或者行为，这就好像你是一个普通公民，但让你穿军装、戴军帽，添加了军人类型的 Introduction 之后，你就拥有军人的特性或者行为。

### Aspect

Aspect 是对系统中的横切逻辑关注点逻辑进行模块化封装的 AOP 概念实体。通常情况下，Aspect 可以包含多个 Pointcut 以及相关 Advice 定义。

> 就是对 Pointcut 和 Advice 进行统一的封装和处理。

### 织入和织入器

织入（Weaving）的过程就是连接 AOP 和 OOP 的桥梁，只有经过织入过程之后，以 Aspect 模块化的横切关注点才会集成到 OOP 的现存系统中。而完成织入这个动作的实体就是织入器。

### 目标对象

符合 Pointcut 所指定的条件，将织入过程中被织入横切逻辑的对象，称为目标对象（Target Object）。

# Spring AOP

Spring AOP 属于第二代 AOP，采用动态代理机制和字节码生成技术实现。动态代理机制和字节码生成都是在运行期间为目标对象生成一个代理对象，而将横切逻辑织入到这个代理对象中，系统最终使用的是织入了横切逻辑的代理对象，而不是真正的目标对象。

## 动态代理机制

动态代理机制的实现主要由一个类和一个接口组成，即 `java.lang.reflect.Proxy` 类和 `java.lang.reflect.InvocationHandler` 接口。使用起来像这样：

```java
/**
 * 代理类与被代理类共同接口
 */
public interface ISubject {
    void request();
}
```

```java
/**
 * 被代理类实现
 */
public class SubjectImpl implements ISubject{
    @Override
    public void request() {
        System.out.println("SubjectImpl::request()");
    }
}
```

```java
/**
 * 动态代理具体实现
 */
public class RequestCtrlInvocationHandler implements InvocationHandler {

   	private final Object target;

   	public RequestCtrlInvocationHandler(Object target) {
      	this.target = target;
   	}

   	/**
     * 横切逻辑的载体，相当于 Advice
     */
   	@Override
   	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    	System.out.println("RequestCtrlInvocationHandler::invoke() begin");
        final Object result = method.invoke(target, args);
        System.out.println("RequestCtrlInvocationHandler::invoke() end");
        return result;
   	}
}
```

```java
/**
 * 使用方式
 */
public static void main(String[] args) {
   	ISubject subject = (ISubject) Proxy.newProxyInstance(
            SubjectImpl.class.getClassLoader(),
            new Class[]{ISubject.class},
            new RequestCtrlInvocationHandler(new SubjectImpl()));
   	subject.request();
}

// 输出：
// RequestCtrlInvocationHandler::invoke() begin
// SubjectImpl::request()
// RequestCtrlInvocationHandler::invoke() end
```

即使还有更多的目标对象类型，只要它们织入的横切逻辑相同，用 RequestCtrlInvocationHandler 一个类并通过 Proxy 为它们生成相应的动态代理实例就可以满足要求。当 Proxy 动态生成的代理对象上相应的接口方法被调用时，对应的 InvocationHandler 就会拦截相应的方法调用，并进行相应处理。

动态代理机制只能对实现了相应 Interface 的类使用，如果某个类没有实现任何的 Interface，就无法使用动态代理机制为其生成相应的动态代理对象。

**为什么动态代理必须实现接口？**

1. 代理模式要求代理类和被代理类需要实现相同的接口；
2. 动态代理生成的代理类会自动继承自 `java.lang.reflect.Proxy`，无法再继承被代理类；
3. `Proxy::newProxyInstance` 方法返回 Object 对象，需要有一个接口可以强制类型转换，才能调用代理方法。

## 字节码生成

使用动态字节码生成技术需要借助 CGLIB 这样的动态字节码生成库，其扩展对象行为的原理是，我们可以对目标对象进行继承扩展，为其生成相应的子类，而子类可以通过重写来扩展父类的行为，只要将横切逻辑的实现放到子类中，然后让系统使用扩展后的目标对象的子类，就可以达到与代理模式相同的结果了。`SubClass instanceof SuperClass == true`。

要使用 CGLIB 进行扩展，首先需要实现一个 `org.springframework.cglib.proxy.Callback`。不过更多的时候，我们会直接使用 `org.springframework.cglib.proxy.MethodInterceptor` 接口（MethodInterceptor 扩展了 Callback 接口）

使用 CGLIB 扩展 SubjectImpl 如下：

```java
public class SubjectCallback implements MethodInterceptor {

    @Override
    public Object intercept(Object obj, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("SubjectCallback::intercept() begin");
        final Object result = methodProxy.invokeSuper(obj, objects);
        System.out.println("SubjectCallback::intercept() end");
        return result;
    }

    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(SubjectImpl.class);
        enhancer.setCallback(new SubjectCallback());

        final SubjectImpl subject = (SubjectImpl) enhancer.create();
        subject.request();
    }

	// 输出：
	// SubjectCallback::intercept() begin
	// SubjectImpl::request()
	// SubjectCallback::intercept() end
}
```

通过为 enhancer 指定需要生成的子类对应的父类，以及 Callback 实现，enhancer 最终为我们生成了需要的代理对象实例。

使用 CGLIB 也有一个缺点，就是**无法对 final 方法进行重写。**

