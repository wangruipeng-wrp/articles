<h1>Spring AOP 实现之 Advice</h1>

> 《Spring揭秘》读书笔记

- [per-class 类型的 Advice](#per-class-类型的-advice)
  - [BeforeAdvice](#beforeadvice)
  - [ThrowsAdvice](#throwsadvice)
  - [AfterReturningAdvice](#afterreturningadvice)
  - [AroundAdvice](#aroundadvice)
- [per-instance 类型的 Advice](#per-instance-类型的-advice)
  - [DelegatingIntroductionInterceptor](#delegatingintroductioninterceptor)
  - [DelegatePerTargetObjectIntroductionInterceptor](#delegatepertargetobjectintroductioninterceptor)

Spring AOP 加入了开源组织 AOP Alliance，目的在于标准化 AOP 的使用，促进各个 AOP 实现产品之间的可交互性。鉴于此，Spring 中的 Advice 实现全部遵循 AOP Alliance 规定的接口。Spring 中各种 Advice 类型实现与 AOP Alliance 中标准接口之间的关系像这样：

![Spring中的Advice略图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Spring中的Advice略图.png)

Advice 实现了将被织入到 Pointcut 规定的 Joinpoint 处的横切逻辑。在 Spring 中，Advice 按照其自身实例能否在目标对象类的所有实例中共享这一标准，可以划分为两大类： `per-class` 类型的 Advice 和 `per-instance` 类型的 Advice。

# per-class 类型的 Advice

该类型的 Advice 实例可以在目标对象类的所有实例之间共享。这种类型的 Advice 通常只是提供方法拦截的功能，不会为目标对象类保存任何状态或者添加新的属性。

## BeforeAdvice

Before Advice 所实现的横切逻辑将在相应的 Joinpoint 之前执行，在 Before Advice 执行完成之后程序执行流程将从 Joinpoint 处继续执行，所以 Before Advice 通常不会打断程序的执行流程。但是如果必要，也可以通过抛出相应异常的形式中断程序流程。

要在 Spring 中实现 Before Advice，通常只需要实现 `org.springframework.aop.MethodBeforeAdvice` 接口即可，该接口定义如下：

```java
public interface MethodBeforeAdvice extends BeforeAdvice {
    void before(Method method, Object[] args, @Nullable Object target) throws Throwable;
}
```

## ThrowsAdvice

Throws Advice 的接口定义在 `org.springframework.aop.ThrowsAdvice`，虽然该接口没有定义任何方法，但是在实现相应的 ThrowsAdvice 时，我们的方法定义需要遵循如下规则：

```java
void afterThrowing([Method, args, target], ThrowableSubclass);
```

其中，`[]` 中的三个参数可以省略。我们可以根据将要拦截的 Throwable 的不同类型。在同一个 ThrowAdvice 中实现多个 afterThrowing 方法。框架会使用 Java 反射机制来调用这些方法。

```java
public class ExceptionBarrierThrowsAdvice implements ThrowsAdvice {
    public void afterThrowing(Throwable t) {
        // 普通异常处理逻辑
    }
    
    public void afterThrowing(RuntimeException e) {
        // 运行时异常处理逻辑
    }
    
    public void afterThrowing(Method m, Object[] args, Object target, ApplicationException e) {
        // 处理应用程序生成的异常
    }
}
```

ThrowsAdvice 通常用于对系统中特定的异常情况进行监控，以统一的方式对所发生的异常进行处理。

## AfterReturningAdvice

`org.springframework.aop.AfterReturningAdvice` 接口定义了 Spring 的 AfterReturningAdvice，如下：

```java
public interface AfterReturningAdvice extends AfterAdvice {
    void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;
}
```

通过 Spring 中的 AfterReturningAdvice，我们可以访问当前 Joinpoint 的方法返回值、方法、方法参数以及所在的目标对象。

如果有需要方法成功执行后进行横切逻辑，使用 AfterReturningAdvice 比较合适。另外，虽然 Spring 的 AfterReturningAdvice 可以访问到方法的返回值，但不可以更改返回值。

## AroundAdvice

Spring 中没有直接定义对应的 Around Advice 接口，而是直接采用 AOP Alliance 的标准接口，即 `org.aopalliance.intercept.MethodInterceptor`，该接口定义如下：

```java
@FunctionalInterface
public interface MethodInterceptor extends Interceptor {
    @Nullable
	Object invoke(@Nonnull MethodInvocation invocation) throws Throwable;
}
```

通过 MethodInterceptor 的 invoke 方法的 MethodInvocation 参数，我们可以控制对相应 Joinpoint 的拦截行为。通过调用 invocation 的 proceed 方法，可以让程序执行继续沿着调用链传播，甚至捕获 proceed 方法可能抛出的异常。如果我们在哪一个 MethodInterceptor 中没有调用 proceed 方法，那么程序的执行将会在当前 MethodInterceptor 处断开执行，Joinpoint 上的调用链将被中断，同一 Joinpoint 上的其他 MethodInterceptor 的逻辑以及 Joinpoint 处的方法逻辑将不会被执行。

> **注意：** 除非清楚当前操作所产生的后果，不然千万不要忘记调用 proceed 方法。

# per-instance 类型的 Advice

与 per-class 类型的 Advice 不同，per-instance 类型的 Advice 不会在目标类所有对象实例之间共享，而是会为不同的实例对象保存它们各自的状态以及相关逻辑。

在 Spring AOP 中，Introduction 就是唯一的一种 per-instance 型 Advice。Introduction 可以在不改动目标类定义的情况下，为目标类添加新的属性以及行为。

在 Spring 中，为目标对象添加新的属性和行为必须声明相应的接口以及相应的实现。这样，再通过特定的拦截器将新的接口定义以及实现类中的逻辑附加到目标对象之上。之后，目标对象（确切地说是目标对象的代理对象）就拥有了新的状态和行为。这个特定的拦截器就是 `org.springframework.aop.IntroductionInterceptor`，其定义如下：

```java
public interface DynamicIntroductionAdvice extends Advice {
	boolean implementsInterface(Class<?> intf);
}

public interface IntroductionInterceptor extends MethodInterceptor, DynamicIntroductionAdvice {

}
```

IntroductionInterceptor 继承了 MethodInterceptor 以及 DynamicIntroductionAdvice。

- 通过 DynamicIntroductionAdvice，我们可以界定当前的 IntroductionInterceptor 为哪些接口提供相应的拦截功能；
- 通过 MethodInterceptor，IntroductionInterceptor 可以处理新添加的接口上的方法调用了。

对于 IntroductionInterceptor 来说如果是新增加的接口上的方法调用，不必去调用 MethodInterceptor 的 proceed 方法，因为当前被拦截的方法实际上就是整个调用链中要最终执行的唯一方法。

因为 Introduction 较之其他 Advice 有些特殊，所以，我们有必要从总体上看一下 Spring 中对 Introduction 的支持结构：

![Introduction相关类结构图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Introduction相关类结构图.png)

Introduction 型的 Advice 两条分支，一条是以 DynamicIntroductionAdvice 为首的动态分支，另一条是以 IntroductionInfo 为首的静态可配置分支。

使用 DynamicIntroductionAdvice，我们可以到运行时再去判定当前 Introduction 可应用到的目标接口类型，而不用预先设定。而 IntroductionInfo 类型则完全相反，其定义如下：

```java
public interface IntroductionInfo {
	Class<?>[] getInterfaces();
}
```

实现类必须返回预定的目标接口类型，这样，在对 IntroductionInfo 型的 Introduction 进行织入的时候，实际上就不需要指定目标接口类型了，因为它自身就带有这些必要的信息。

大多数时候，我们如果要对目标对象拦截并添加 Introduction 逻辑，可以直接使用 Spring 提供的两个现成的实现类就可以了。

## DelegatingIntroductionInterceptor

DelegatingIntroductionInterceptor 不会自己实现将要添加到目标对象上的新的逻辑行为，而是委派给其他实现类，用法像这样：

> 以简化开发人员为例，有时候测试人员忙不过来了，可能就会要几个开发人员过去帮忙做测试，那么这个时候原来的开发人员就拥有了测试人员的行为和属性。

```java
public interface IDeveloper {
    void developSoftware();
}

public class Developer implements IDeveloper {
    @Override
    public void developSoftware() {
        System.out.println("I am happy with programming");
    }
}
```

要为 Developer 添加测试人员的职能，要先将测试人员的职能定义成接口，像这样：

```java
public interface ITester {
    boolean isBusyAsTester();
    void testSoftware();
}
```

然后要给出测试人员的实现类，在实现类中给出要添加到目标对象（开发人员）的具体逻辑。当目标对象将要行使新的职能的时候，会通过该实现类寻求帮助：

```java
public class Tester implements ITester {
    private boolean busyAsTester;

    @Override
    public void testSoftware() {
        System.out.println("I will ensure the quality");
    }

    @Override
    public boolean isBusyAsTester() {
        return busyAsTester;
    }

    public void setBusyAsTester(boolean busyAsTester) {
        this.busyAsTester = busyAsTester;
    }
}
```

通过 DelegatingIntroductionInterceptor 进行 Introduction 的拦截。有了新增加职能的接口定义以及相应实现类，使用 DelegatingIntroductionInterceptor，我们就可以把具体的 Inroduction 拦截委托给具体的实现类来完成，像这样：

```java
ITester delegate = new Tester();
DelegatingIntroductionInterceptor interceptor = new DelegatingIntroductionInterceptor(delegate);

// 给 developer 增加 tester 的职能
IDeveloper developer = new Developer();
ITester tester = (ITester) weaver.weave(developer).with(interceptor).getProxy();
tester.testSoftware();
```

DelegatingIntroductionInterceptor 虽然是 Introduction 型的 Advice 实现，但它并没有兑现 Inroduction 作为 per-instance 型 Advice 的承诺。因为它只持有一个 “delegate”，供同一目标对象的所有实例共享使用，做不到为不同的实例对象保存它们各自的状态以及相关逻辑。

## DelegatePerTargetObjectIntroductionInterceptor

与 DelegatingIntroductionInterceptor 不同 DelegatePerTargetObjectIntroductionInterceptor 会在内部持有一个目标对象与相应的 Introduction 逻辑实现类之间的映射关系。当每个目标对象上的新定义的接口方法被调用的时候，DelegatePerTargetObjectIntroductionInterceptor 会拦截这些调用，然后以目标对象实例作为键，到它持有的哪个映射关系中取得对应当前目标对象实例的 Introduction 实现类实例。

以此，DelegatePerTargetObjectIntroductionInterceptor 可以完全实现对 per-instance 的承诺。

与 DelegatingIntroductionInterceptor 的使用没有太大的区别，主要体现在构造方式上的不同。现在我们不需要自己构造 delegate 接口实例，而是只需要告知 DelegatePerTargetObjectIntroductionInterceptor 相应的 delegate 接口类型和对象实现类的类型，像这样：

```java
DelegatePerTargetObjectIntroductionInterceptor interceptor = new DelegatePerTargetObjectIntroductionInterceptor(Tester.class, ITester.class);
```

---

最后要说的是 Introduction 的性能问题。与 AspectJ 直接通过编译器将 Inroduction 织入目标对象不同，Spring AOP 采用的是动态代理机制，在性能上，Inroduction 型的 Advice 要差一些。如果有必要，可以考虑采用 AspectJ 的 Introduction 实现。
