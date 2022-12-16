---
title: Spring AOP 为 目标对象注入代理
abbrlink: 268
date: 2022-12-14 20:57:28
description: 《Spring揭秘》读书笔记
---

使用 Spring AOP 的过程中可能会出现这样一个问题：为目标对象创建代理的时候，如果目标对象上的某些方法依赖于自身逻辑，而这部分被依赖逻辑又被刚好是一个 Joinpoint 被 Pointcut 捕捉到了，那在对象内部调用这部分逻辑时，这次调用是没办法被捕捉到的，这也是代理模式的一个小的缺陷。下面演示一下这个问题：

这是目标对象，即将对它生成代理，Joinpoint 是 method2：

```java
@Component
public class TargetObj {
    public void method1() {
        method2();
        System.out.println("method1 executed");
    }

    public void method2() {
        System.out.println("method2 executed");
    }
}
```

使用 @AspectJ 对 TargetObj 生成代理：

```java
@Aspect
@Component
public class TargetObjAspect {

    @Pointcut("execution(public void TargetObj.method1())")
    public void method1() { }

    @Pointcut("execution(public void TargetObj.method2())")
    public void method2() { }

    @Around("method1() || method2()")
    public Object aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("start...");
        Object res = joinPoint.proceed();
        System.out.println("end...");
        return res;
    }
}

```

Spring Boot 的启动类：

```java
@SpringBootApplication
@EnableAspectJAutoProxy
public class SpringDemoApplication {

    public static void main(String[] args) {
        final ApplicationContext ac = SpringApplication.run(SpringDemoApplication.class, args);
        ac.getBean("target", TargetObj.class).method1();
    }
}
```

直接调用 method1 方法，在 method1 方法中调用的 method2 方法并不会被拦截，只有 method1 方法被拦截，其原因是由 Spring AOP 的实现机制造成的。

我们知道，Spring AOP 采用代理模式实现 AOP，具体的横切逻辑会被添加到动态生成的代理对象中，只要调用的是目标对象的代理对象上的方法，通常就可以保证目标对象上的方法执行可以被拦截。不过，代理模式的实现机制在处理方法调用的时序方面，会给使用这种机制的 AOP 产品造成一定的影响，如下所示：

```java
proxy.method1() {
    System.out.println("start...");
    target.method2();
    System.out.println("end...");
}
```

在代理对象方法中，不管如何添加横切逻辑，你终归都是需要调用目标对象上的同一方法来执行最初所定义的方法逻辑。此时，如果目标对象中原始方法依赖于其他对象，那我们可以为目标对象注入所依赖对象的代理，这可以保证相应的 Joinpoint 被拦截并且织入横切逻辑。

但如果目标对象调用的是自身的方法，也就是依赖于自身所定义的其他方法的时候，就得把自身的代理对象注入到自身当中，才可以保证相应的 Joinpoint 被拦截并注入横切逻辑。

上面的例子中，method1 直接调用了 method2，这次的调用就没有走代理对象的调用，而是直接调用了自身，导致此次调用没有被拦截。

解决方法就是将代理对象注入当前对象，不再去调用自身方法的时候，而是调用代理对象的方法。这样在代理对象中的 target 就变成了 this：

```java
proxy.method1() {
    System.out.println("start...");
    this.method2();
    System.out.println("end...");
}
```

Spring AOP 提供了 AopContext 来公开当前目标对象的代理对象，我们只要在目标对象中使用 `AopContext.currentProxy()` 就可以取得当前目标对象所对应的代理对象。现在，我们重构 method1 方法，让它直接调用它的代理对象的 method2 方法：

```java
public void method1() {
    System.out.println("method1 executed");
    ((TargetObj) AopContext.currentProxy()).method2();
}
```

要使 `AopContext.currentProxy()` 生效，我们在生成目标对象的代理对象时，需要将 ProxyConfig 或者它的相应子类的 exposeProxy 属性设置为 true，在 Spring Boot 中就只用在启动类上的 @EnableAspectJAutoProxy 注解加上这个参数：`@EnableAspectJAutoProxy(exposeProxy = true)`

现在再执行一次就能成功拦截到 method2 方法了。

在 Spring 中，通常可以借助 IoC 容器来注入  `AopContext.currentProxy()` 对象。可以声明一个 Wrapper 接口，并让目标对象实现这个接口，再通过 BeanPostProcessor 来注入。