<h1>Spring AOP 实现之 Aspect</h1>

> 《Spring揭秘》读书笔记

- [PointcutAdvisor](#pointcutadvisor)
  - [DefaultPointcutAdvisor](#defaultpointcutadvisor)
  - [NameMatchMethodPointcutAdvisor](#namematchmethodpointcutadvisor)
  - [RegexpMethodPointcutAdvisor](#regexpmethodpointcutadvisor)
  - [DefaultBeanFactoryPointcutAdvisor](#defaultbeanfactorypointcutadvisor)
- [IntroductionAdvisor](#introductionadvisor)
- [Ordered](#ordered)

Advisor 代表 Spring 中的 Aspect，但是与正常的 Aspect 不同，Advisor 通常只持有一个 Pointcut 和一个 Advice。理论上 Aspect 定义中可以有多个 Pointcut 和 Advice，所以，我们可以认为 Advisor 是一种特殊的 Aspect。

为了能够更清楚 Advisor 的实现结构体系，可以将 Advisor 简单划分为两个分支，一个分支以 `org.springframework.aop.PointcutAdvisor` 为首，另一个分支以 `org.springframework.aop.IntroductionAdvisor` 为首。

# PointcutAdvisor

实际上，PointcutAdvisor 才是真正定义一个 Pointcut 和一个 Advice 的 Advisor，大部分的 Advisor 实现都是 PointcutAdvisor 的实现类。

![PointcutAdvisor及相关子类](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/PointcutAdvisor及相关子类.png)

## DefaultPointcutAdvisor

最通用的 PointcutAdvisor 实现，除了不能为其指定 Introduction 类型的 Advice 之外，剩下的 任何类型的 Pointcut、任何类型的 Advice 都可以通过它来使用。使用起来像这样：

```java
Pointcut pointcut = ...; // 任何 Pointcut 类型
Advice   advice   = ...; // 除 Inroduction 类型外的任何 Advice 类型

DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, advice);
// 可以通过构造器或者 setter 注入 pointcut 和 advice
```

## NameMatchMethodPointcutAdvisor

限定了自身可以使用的 Pointcut 类型为 NameMatchMethodPointcut，且外部不可更改。不过，对于使用的 Advice 来说，除了 Introduction，其他任何类型的 Advice 都可以使用。使用起来像这样：

```java
Advice advice = ...; // 除 Inroduction 类型外的任何 Advice 类型

NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor(advice);
advisor.setMappedName("methodName");
// OR
advisor.setMappedNames(new String[]{"methodName1", "methodName2"});
```

## RegexpMethodPointcutAdvisor

限定了自身可以使用的 Pointcut 的类型，即只能通过正则表达式为其设置相应的 Pointcut。

其自身内部持有一个 AbstractRegexpMethodPointcut 的实例。AbstractRegexpMethodPointcut 有两个子类，即 JdkRegexpMethodPointcut 和 Perl5RegexpMethodPointcut。默认会使用 JdkRegexpMethodPointcut，如果要强制使用 Perl5RegexpMethodPointcut 可以通过 setPerl5(boolean) 方法设置。

## DefaultBeanFactoryPointcutAdvisor

使用较少，其使用必须绑定 IoC 容器，作用是可以通过容器中注册的 Advice 注册的 BeanName 来关联对应的 Advice。

# IntroductionAdvisor

IntroductionAdvisor 只能应用于类级别的拦截，只能使用 Inroduction 型的 Advice，纯粹就是为了 Inroduction 而生的。只有一个默认实现：DefaultIntroductionAdvisor。使用起来也比较简单，只可以指定 Inroduction 型的 Advice（即 IntroductionInterceptor）以及将被拦截的接口类型。

# Ordered

Spring 在处理同一 Joinpoint 处的多个 Advisor 的时候，实际上会按照指定的顺序和优先级来执行它们，顺序号决定优先级，顺序号越小，优先级越高，优先级排在前面的，将被优先执行。我们可以从0或者1开始指定，因为小于的顺序号原则上由 Spring AOP 框架内部使用。默认情况下，如果我们不明确指定各个 Advisor 的执行顺序，那么 Spring 会按照它们的声明顺序来应用它们，最先声明的顺序号最小但优先级最大，其次次之。
