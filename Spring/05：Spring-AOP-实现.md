---
title: Spring AOP 实现
abbrlink: 22032
date: 2022-12-04 11:46:38
description: 《Spring揭秘》读书笔记
---

# Joinpoint

Spring AOP 中，仅支持方法执行类型的 Joinpoint。也就是仅能在方法内部执行开始时点进行横切逻辑织入。

# Pointcut

Spring 中以接口定义 `org,springframework.aop.Pintcut` 作为其 AOP 框架中所有 Pointcut 的最顶层抽象，该接口定义了两个方法用来帮助捕捉系统中的相应 Joinpoint，并提供了一个 TruePointcut 类型实例。如果 Pointcut 类型为 TruePointcut ，默认会对系统中的所有对象，以及对象上所有被支持的 Joinpoint 进行匹配。`org.springframeworkaop.Pointcut` 接口定义如下:

```java
public interface Pointcut {
    ClassFilter getClassFilter();
    MethodMatcher getMethodMatcher();
    Pointcut TRUE = TruePointcut.INSTANCE;
}
```

## ClassFilter

ClassFilter 和 MethodMatcher 分别用于匹配将被执行织入操作的对象以及相应的方法。ClassFilter 接口的作用是对 Joinpoint 所处的对象进行 Class 级别的类型匹配，其定义如下：

```java
public interface ClassFilter {
    boolean matches(Class clazz);
    ClassFilter TRUE = TrueClassFilter.INSTANCE;
}
```

当织入的目标对象的 Class 类型与 Pointcut 所规定的类型相符时，matches 方法将会返回 true，否则，返回 false，即意味着不会对这个类型的目标对象进行织入操作。

```java
/**
 * 如果希望对系统中 Foo 类型的类进行织入，则可以这么来定义 ClassFilter
 */
public class FooClassFilter implements ClassFilter {
    @Override
    public boolean matches(Class<?> cl) {
        return Foo.class.equals(cl);
    }
}
```

当然，如果类型对我们所捕获的 Joinpoint 无所谓，那么 Pointcut 中使用的 ClassFilter 可以直接返回 `TrueClassFilter.INSTANCE`。当 Pointcut 中返回的 ClassFilter 类型为该类型实例时，Pointcut 的匹配将会针对系统中所有的目标类以及它们的实例进行。

## MethodMatcher

```java
public interface MethodMatcher {
	boolean matches(Method method, Class<?> targetClass);
	boolean isRuntime();
	boolean matches(Method method, Class<?> targetClass, Object... args);
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
```

MethodMatcher 定义了两个 matches 方法，这两个方法的分界线就是 isRuntime 方法。在对对象具体方法进行拦截的时候，可以忽略每次方法执行的时候调用者传入的参数，也可以每次都检查这些方法调用参数，以强化拦截条件。

- **isRuntime()：**
- **返回 false：**表示不会考虑具体 Joinpoint 的方法参数，这种类型的 MethodMatcher 称为 `StaticMethodMatcher`。因为不用每次都检查参数，所以只有 `boolean matches(Method method, Class<?> targetClass);` 会被执行，那么对于同样类型的方法匹配结果，就可以在框架内部缓存以提高性能。
- **返回 true：**表示该 MethodMatcher 将会每次都对方法调用的参数进行匹配检查，这种类型的 MethodMatcher 称之为 `DynamicMethodMatcher`，匹配效率相对于 `StaticMethodMatcher` 来说要差一些，最好避免使用。当 isRuntime 返回 true 后，并且当 `boolean matches(Method method, Class<?> targetClass);` 方法也返回 true 的时候，三个参数的 matches 方法才会被执行，以进一步检查匹配条件。如果两个参数的 matches 方法返回 false，那么不管 isRuntime 方法返回 true 还是 false，都不会执行三个参数 matches 方法。

各个 Pointcut 类型之间的关系图：

![Pointcut类型之间的关系图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Pointcut类型之间的关系图.png)

### NameMatchMethodPointcut

这是最简单的 Pointcut 实现，可以根据自身指定的一组方法名称与 Joinpoint 处的方法名称进行匹配。除了可以指定方法名，还可以使用 “*” 通配符，实现简单的模糊匹配。

但是，NameMatchMethodPointcut 无法对重载的方法名进行匹配，因为它仅对方法名进行匹配，不会考虑参数相关信息，而且也没有提供可以指定参数匹配信息的途径。

### JdkRegexpMethodPointcut和Perl5RegexpMethodPointcut

通过正则表达式匹配方法名。

### AnnotationMatchingPointcut

根据目标对象中是否存在指定类型的注解来匹配 Joinpoint，要使用该类型的 Pointcut，首先需要生命相应的注解。

```java
/**
 * 类级别的注解
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ClassLevelAnnotation {
}

/**
 * 方法级别的注解
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MethodLevelAnnotation {
}
```

使用起来像这样：

```java
AnnotationMatchingPointcut pointcut = new AnnotationMatchingPointcut(ClassLevelAnnotation.class, MethodLevelAnnotation.class);
```

### ComposablePointcut

Pointcut 通常还提供逻辑运算功能，而 ComposablePointcut 就是 Spring AOP 提供的可以进行 Pointcut 逻辑运算的 Pointcut 实现。像这样：

```java
ComposablePointcut pointcut1 = new ComposablePointcut(classFilterl,methodMatcher1);
ComposablePointcut pointcut2 = ...

ComposablePointcut unitedPoincut= pointcutl.union(pointcut2);
ComposablePointcut intersectionPointcut = pointcutl,intersection(unitedPoincut);

assertEquals(pointcutl,intersectionPointcut);
```

也能通过 `org.springframework.aop.support.Pointcuts` 工具类来实现：

```java
Pointcut pointcut1 = ...
Pointcut pointcut2 = ...

Pointcut unitedPoincut          = Pointcuts.union(pointcut1,pointcut2);
Pointcut intersectionPointcut   = Pointcuts.intersection(pointcutl,pointcut2);
```

### ControlFlowPointcut

ControlFlowPointcut 匹配程序的调用流程，不是对某个方法执行所在的 Joinpoint 处的单一特征进行匹配。

假设我们所拦截的目标对象如下：

```java
public class TargetObject {
    public void method1() {
        // ...
    }
}
```

它的调用类是：

```java
public class TargetCaller {
    private TargetObject target;

    public void callMethod() {
        target.method1();
    }

    public void setTarget(TargetObject target) {
        this.target = target;
    }
}
```

如果使用之前的任何 Pointcut 实现，我们只能指定在 Targetobject 的 method1 方法每次执行的时候，都织入相应横切逻辑。也就是说，一旦通过 Pointcut 指定 method1 处为 Joinpoint，那么对该方法的执行进行拦截是必定的，不管 method1 是被谁调用。而通过 controlFlowPointcut，我们可以指定，只有当 Targetobject 的 method1 方法在 TargetCaller 类所声明的方法中被调用的时候，才对 method1 方法进行拦截，其他地方调用 method1 的话，不对 method1 进行拦截。

ControlFlowPointcut 使用起来大概像这样：

```java
ControlFlowPointcut pointcut = new ControlFlowPointcut(TargetCaller.class);
Advice advice = ... ;

TargetObject target = new TargetObject();
TargetObject targetProxy = weaver.weave(advice).to(target).accordingto(pointcut);

// advice 的逻辑在这里不会被触发执行
targetProxy.method1();

// advice 的逻辑在这里不会被触发执行
// 因为 TargetCaller 的 callMethod() 将调用 method1()
// 正像 ControlFlowPointcut(TargetCaller.class) 所指定的那样
TargetCaller caller = new TargetCaller();
caller.setTarget(targetProxy);
caller.callMethod();
```

当织入器按照 Pointcut 的规定，将 Advice 织入到目标对象之后，从任何其他地方调用 method1，是不会触发 Advice 所包含的横切逻辑执行的。只有在 ControlFlowPointcut 规定的类内部调用目标对象的 method1，才会触发 Advice 中横切逻辑的执行。

> **注意：**示例代码中 weaver 是虚构概念，不是 Spring 织入器的实现。

如果在 ControlFlowPointcut 的构造方法中单独指定 Class 类型的参数，那么 ControlFlowPointcut 将尝试匹配指定的 Class 中声明的所有方法，跟目标对象的 Joinpoint 处的方法流程组合。如果还想指定调用类的某个具体方法，可以在构造 ControlFlowPointcut 的时候，传入第二个参数，即调用方法的名称：

```java
ControlFlowPointcut pointcut = new ControlFlowPointcut(TargetCaller.class, "callMethod");
```

### 自定义 StaticMethodMatcherPointcut

> 继承抽象类 StaticMethodMatcher 即可。

StaticMethodMatcher 根据自身语意，为其子类提供了如下几个方面的默认实现。

- 默认所有 StaticMethodMatcher 的子类的 ClassFilter 均为 `ClassFilter.TRUE` 即忽略类的类型匹配。如果具体子类需要对目标对象的类型做进一步限制，可以通过 `public void setclassFilter(ClassFilter classFilter)` 方法设置相应的 ClassFilter 实现。
- 因为是 staticMethodMatcher，所以，其 MethodMatcher 的 isRuntime 方法返回 false，同时三个参数的 matches 方法抛出 `UnsupportedOperationException` 异常，以表示该方法不应该被调用到。

最终我们需要做的就是实现两个参数的 matches 方法了

### 自定义 DynamicMethodMatcherPointcut

> 继承抽象类 DynamicMethodMatcher 即可。

DynamicMethodMatcherPointcut 也为其子类提供了部分便利。

- getclassFilter() 方法返回 `ClassFilter.TRUE`，如果需要对特定的目标对象类型进行限定子类只要覆写这个方法即可。
- 对应的 MethodMatcher 的 isRuntime 总是返回 true，同时，StaticMethodMatcherPointcut 提供了两个参数的 matches 方法的实现，默认直接返回 true。

要实现自定义 DynamicMethodMatcherPointcut，通常情况下，我们只需要实现三个参数的 matches 方法逻辑即可。但如果愿意，我们也可以覆写一下两个参数的 matches 方法，这样，不用每次都得到三个参数的 matches 方法执行的时候才检查所有的条件。

# Advice

Spring AOP 加入了开源组织 AOP Alliance，目的在于标准化 AOP 的使用，促进各个 AOP 实现产品之间的可交互性。鉴于此，Spring 中的 Advice 实现全部遵循 AOP Alliance 规定的接口。Spring 中各种 Advice 类型实现与 AOP Alliance 中标准接口之间的关系像这样：

![Spring中的Advice略图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Spring中的Advice略图.png)

Advice 实现了将被织入到 Pointcut 规定的 Joinpoint 处的横切逻辑。在 Spring 中，Advice 按照其自身实例能否在目标对象类的所有实例中共享这一标准，可以划分为两大类，即 `per-class` 类型的 Advice 和 `per-instance` 类型的 Advice。

## per-class 类型的 Advice

该类型的 Advice 实例可以在目标对象类的所有实例之间共享。这种类型的 Advice 通常只是提供方法拦截的功能，不会为目标对象类保存任何状态或者添加新的属性。

### BeforeAdvice

Before Advice 所实现的横切逻辑将在相应的 Joinpoint 之前执行，在 Before Advice 执行完成之后程序执行流程将从 Joinpoint 处继续执行，所以 Before Advice 通常不会打断程序的执行流程。但是如果必要，也可以通过抛出相应异常的形式中断程序流程。

要在 Spring 中实现 Before Advice，通常只需要实现 `org.springframework.aop.MethodBeforeAdvice` 接口即可，该接口定义如下：

```java
public interface MethodBeforeAdvice extends BeforeAdvice {
    void before(Method method, Object[] args, @Nullable Object target) throws Throwable;
}
```

### ThrowsAdvice

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
    
    public void afterThrowing(Method m, Object[] args, 0bject target, ApplicationException e) {
        // 处理应用程序生成的异常
    }
}
```

ThrowsAdvice 通常用于对系统中特定的异常情况进行监控，以统一的方式对所发生的异常进行处理。

### AfterReturningAdvice

`org.springframework.aop.AfterReturningAdvice` 接口定义了 Spring 的 AfterReturningAdvice，如下：

```java
public interface AfterReturningAdvice extends AfterAdvice {
    void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;
}
```

通过 Spring 中的 AfterReturningAdvice，我们可以访问当前 Joinpoint 的方法返回值、方法、方法参数以及所在的目标对象。

如果有需要方法成功执行后进行横切逻辑，使用 AfterReturningAdvice 比较合适。另外，虽然 Spring 的 AfterReturningAdvice 可以访问到方法的返回值，但不可以更改返回值。

### AroundAdvice

Spring 中没有直接定义对应的 Around Advice 接口，而是直接采用 AOP Alliance 的标准接口，即 `org.aopalliance.intercept.MethodInterceptor`，该接口定义如下：

```java
@FunctionalInterface
public interface MethodInterceptor extends Interceptor {
    @Nullable
	Object invoke(@Nonnull MethodInvocation invocation) throws Throwable;
}
```

通过 MethodInterceptor 的 invoke 方法的 MethodInvocation 参数，我们可以控制对相应 Joinpoint 的拦截行为。通过调用 invocation 的 proceed 方法，可以让程序执行继续沿着调用链传播，甚至捕获 proceed 方法可能抛出的异常。如果我们在哪一个 MethodInterceptor 中没有调用 proceed 方法，那么程序的执行将会在当前 MethodInterceptor 处断开执行，Joinpoint 上的调用链将被中断，同一 Joinpoint 上的其他 MethodInterceptor 的逻辑以及 Joinpoint 处的方法逻辑将不会被执行。

> **注意：**除非清楚当前操作所产生的后果，不然千万不要忘记调用 proceed 方法。

## per-instance 类型的 Advice

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

### DelegatingIntroductionInterceptor

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

### DelegatePerTargetObjectIntroductionInterceptor

与 DelegatingIntroductionInterceptor 不同 DelegatePerTargetObjectIntroductionInterceptor 会在内部持有一个目标对象与相应的 Introduction 逻辑实现类之间的映射关系。当每个目标对象上的新定义的接口方法被调用的时候，DelegatePerTargetObjectIntroductionInterceptor 会拦截这些调用，然后以目标对象实例作为键，到它持有的哪个映射关系中取得对应当前目标对象实例的 Introduction 实现类实例。

以此，DelegatePerTargetObjectIntroductionInterceptor 可以完全实现对 per-instance 的承诺。

与 DelegatingIntroductionInterceptor 的使用没有太大的区别，主要体现在构造方式上的不同。现在我们不需要自己构造 delegate 接口实例，而是只需要告知 DelegatePerTargetObjectIntroductionInterceptor 相应的 delegate 接口类型和对象实现类的类型，像这样：

```java
DelegatePerTargetObjectIntroductionInterceptor interceptor = new DelegatePerTargetObjectIntroductionInterceptor(Tester.class, ITester.class);
```

---

最后要说的是 Introduction 的性能问题。与 AspectJ 直接通过编译器将 Inroduction 织入目标对象不同，Spring AOP 采用的是动态代理机制，在性能上，Inroduction 型的 Advice 要差一些。如果有必要，可以考虑采用 AspectJ 的 Introduction 实现。

# Aspect

Advisor 代表 Spring 中的 Aspect，但是与正常的 Aspect 不同，Advisor 通常只持有一个 Pointcut 和一个 Advice。理论上 Aspect 定义中可以有多个 Pointcut 和 Advice，所以，我们可以认为 Advisor 是一种特殊的 Aspect。

为了能够更清楚 Advisor 的实现结构体系，可以将 Advisor 简单划分为两个分支，一个分支以 `org.springframework.aop.PointcutAdvisor` 为首，另一个分支以 `org.springframework.aop.IntroductionAdvisor` 为首。

## PointcutAdvisor

实际上，PointcutAdvisor 才是真正定义一个 Pointcut 和一个 Advice 的 Advisor，大部分的 Advisor 实现都是 PointcutAdvisor 的实现类。

![PointcutAdvisor及相关子类](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/PointcutAdvisor及相关子类.png)

### DefaultPointcutAdvisor

最通用的 PointcutAdvisor 实现，除了不能为其指定 Inroduction 类型的 Advice 之外，剩下的 任何类型的 Pointcut、任何类型的 Advice 都可以通过它来使用。使用起来像这样：

```java
Pointcut pointcut = ...; // 任何 Pointcut 类型
Advice   advice   = ...; // 除 Inroduction 类型外的任何 Advice 类型

DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, advice);
// 可以通过构造器或者 setter 注入 pointcut 和 advice
```

### NameMatchMethodPointcutAdvisor

限定了自身可以使用的 Pointcut 类型为 NameMatchMethodPointcut，且外部不可更改。不过，对于使用的 Advice 来说，除了 Inroduction，其他任何类型的 Advice 都可以使用。使用起来像这样：

```java
Advice advice = ...; // 除 Inroduction 类型外的任何 Advice 类型

NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor(advice);
advisor.setMappedName("methodName");
// OR
advisor.setMappedNames(new String[]{"methodName1", "methodName2"});
```

### RegexpMethodPointcutAdvisor

限定了自身可以使用的 Pointcut 的类型，即只能通过正则表达式为其设置相应的 Pointcut。

其自身内部持有一个 AbstractRegexpMethodPointcut 的实例。AbstractRegexpMethodPointcut 有两个子类，即 JdkRegexpMethodPointcut 和 Perl5RegexpMethodPointcut。默认会使用 JdkRegexpMethodPointcut，如果要强制使用 Perl5RegexpMethodPointcut 可以通过 setPerl5(boolean) 方法设置。

### DefaultBeanFactoryPointcutAdvisor

使用较少，其使用必须绑定 IoC 容器，作用是可以通过容器中注册的 Advice 注册的 BeanName 来关联对应的 Advice。

## IntroductionAdvisor

IntroductionAdvisor 只能应用于类级别的拦截，只能使用 Inroduction 型的 Advice，纯粹就是为了 Inroduction 而生的。只有一个默认实现：DefaultIntroductionAdvisor。使用起来也比较简单，只可以指定 Inroduction 型的 Advice（即 IntroductionInterceptor）以及将被拦截的接口类型。

## Ordered

Spring 在处理同一 Joinpoint 处的多个 Advisor 的时候，实际上会按照指定的顺序和优先级来执行它们，顺序号决定优先级，顺序号越小，优先级越高，优先级排在前面的，将被优先执行。我们可以从0或者1开始指定，因为小于的顺序号原则上由 Spring AOP 框架内部使用。默认情况下，如果我们不明确指定各个 Advisor 的执行顺序，那么 Spring 会按照它们的声明顺序来应用它们，最先声明的顺序号最小但优先级最大，其次次之。

# 织入

在 Spring AOP 中，使用类 `org.springframework.aop.framework.ProxyFactory` 作为织入器。ProxyFactory 并非 Spring AOP 中唯一可用的织入器，而是最基本的一个织入器实现。

Spring AOP 是基于代理模式的 AOP 实现，织入过程完成后，会返回织入了横切逻辑的目标对象的代理对象。使用 ProxyFactory 只需要指定如下两个最基本的东西：

- **要对其进行织入的目标对象。**可以通过 ProxyFactory 的构造器或者 setter 注入。
- **将要应用到目标对象的 Aspect，也就是 Spring 里面的 Advisor。**
  - 对于 Inroduction 之外的 Advice 类型，ProxyFactory 内部就会为这些 Advice 构造相应的 Advisor，只不过在为它们构造的 Advisor 中使用的 Pointcut 为 `Pointcut.TRUE`，这些 Advice 将被应用到系统中所有可识别的 Joinpoint 处。
  - 如果要添加的 Advice 类型是 Inroduction 类型，**则会根据该 Inroduction 的具体类型进行区分：**
    - IntroductionInfo：因为它本身包含了必要的描述信息，框架内部会为其构造一个 DefaultIntroductionAdvisor；
    - DynamicIntroductionAdvice：框架内部将抛出 AopConfigException 异常（因为无法从DynamicIntroductionAdvice取得必要的目标对象信息）

## ProxyFactory 使用

```java
public interface ITask {
    void execute();
}

public class Task implements ITask {
    @Override
    public void execute() {
        System.out.println("execute");
    }
}
```

### 基于接口的代理

```java
// 被代理对象
Task task = new Task();

// 构建 Advisor
NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor();
advisor.setMappedName("execute");
Advice advice = ...;
advisor.setAdvice(advice);

// 织入
ProxyFactory weaver = new ProxyFactory(task);
weaver.addAdvisor(advisor);
weaver.setInterface(new Class[]{ITask.class});

// 使用代理对象
ITask proxyTask = (ITask) weaver.getProxy();
proxyTask.execute();
```

通过 setInterface 方法可以明确告知 ProxyFactory，我们要对 ITask 接口类型进行代理。不过，如果没有其他行为属性的干预，我们也可以不使用 setInterface 方法明确指定具体的接口类型。默认情况下，ProxyFactory 只要检测到目标类实现了相应的接口，也会对目标类进行基于接口的代理。

简单点儿说，如果目标类实现了至少一个接口，不管我们有没有通过 ProxyFactory 的setInterfaces 方法明确指定要对特定的接口类型进行代理，只要不将 ProxyFactory 的 `optimize` 和 `proxyTargetclass` 两个属性的值设置为 true (这两个属性稍后将谈到)，那么 ProxyFactory 都会按照面向接口进行代理。

### 基于类的代理

如果目标类没有实现任何接口，那么，默认情况下，ProxyFactory 会对目标类进行基于类的代理，即使用 CGLIB。假设现在有一个对象，如下：

```java
public class Executable {
    public void execute() {
        System.out.println("Executable without any Interfaces");
    }
}
```

对 Executable 进行代理的过程像这样：

```java
// 省略构建 advisor

// 被代理对象
Executable executable = new Executable();

// 织入
ProxyFactory weaver = new ProxyFactory(executable);
weaver.addAdvisor(advisor)

// 使用代理对象
Executable executable = (Executable) weaver.getProxy(); 
```

但是，即使目标对象类实现了至少一个接口，我们也可以通过 proxyTargetClass 属性强制 ProxyFactory 采用基于类的代理。以 Task 为例，它实现了 ITask 接口，默认情况下 ProxyFactory 对其会采用基于接口的代理，但是，通过 proxyTargetClass，我们可以改变这种默认行为，像这样：

```java
weaver.setProxyTargetClass(true);
```

除此之外，如果将 ProxyFactory 的 optimize 属性设定为 true 的话，ProxyFactory 也会采用基于类的代理机制。总的来说，如果满足一下列出的三种情况中的任何一种，ProxyFactory 将对目标类进行基于类的代理：

1. 目标类没有实现任何接口
2. ProxyFactory 的 proxyTargetClass 属性被设置为 true
3. ProxyFactory 的 optimeize 属性被设置为 true

### Introduction 的织入

Introduction 型的 Advice 比较特殊，如下所述：

- Introduction 可以为已经存在的对象类型添加新的行为，只能应用于对象级别的拦截，而不是通常 Advice 的方法级别的拦截，所以，进行 Inroduction 的织入过程中，不需要指定 Pointcut，而是需要指定目标接口类型。
- Spring 的 Introduction 支持只能通过接口定义为当前对象添加心的行为，所以，我们需要在织入的时机，指定新织入的接口类型。

```java
// 织入对象
IDeveloper developer = new Developer();
ProxyFactory weaver = new ProxyFactory(developer);
weaver.setInterface(new Class[]{IDeveloper.class, ITester.class});

// 织入逻辑
Advice advice = ...;
DefaultIntroductionAdvisor advisor = new DefaultIntroductionAdvisor(advice);
weaver.addAdvisor(advisor);

// 使用代理对象
Object proxy = weaver.getproxy();
((ITester) proxy).testSoftware();
((IDeveloper) proxy).developSoftware();
```

对 Introduction 进行织入，新添加的接口类型必须是通过 setInterfaces 指定的。而原来的目标对象，是采用基于接口的代理形式还是采用基于类的代理形式，完全是可以自由选择的。

## ProxyFactory 实质

为 ProxyFactory 提供具体代理机制实现的接口：`org.springframework.aop.framework.AopProxy`

```java
public interface AopProxy {
	Object getProxy();
	Object getProxy(@Nullable ClassLoader classLoader);
}
```

Spring AOP 框架内使用 AopProxy 对使用的不同的代理实现机制进行了适度的抽象，针对不同的代理实现机制提供相应的 AopProxy 子类实现。目前，Spring AOP 框架内提供了针对 JDK 的动态代理和 CGLIB 两种机制的 AopProxy 实现。

![AopProxy相关结构图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/AopProxy相关结构图.png)

### AopProxyFactory

因为动态代理需要通过 InvocationHndler 提供调用拦截，所以 JdkDynamicAopProxy 同时实现了 InvocationHndler 接口。不同 AopProxy 实现的实例化过程采用工厂模式（确切地说是抽象工厂模式）进行封装，即通过 `org.springframework.aop.framework.AopProxyFactory` 进行。其定义如下：

```java
public interface AopProxyFactory {
    AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;
}
```

AopProxyFactory 根据传入的 AdvisedSupport 实例提供的相关信息，来决定生成什么类型的 AopProxy。不过具体的工作会转交给它的实现类：`org.springframework.aop.framework.DefaultAopProxyFactory`。它的工作逻辑很简单，就是判断是否采用 CGLIB 生成动态代理对象，满足直接生成，否则生成 JDK 动态代理对象。

### AdvisedSupport

AdvisedSupport 其实就是一个生成代理对象所需要的信息的载体，该类相关的类层次图如下：

![AdvisedSupport类层次图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/AdvisedSupport类层次图.png)

AdvisedSupport 所承载的信息可以划分为两类，一类以 `org.springframework.aop.framework.ProxyConfig` 为统领，记载生成代理对象的控制信息；一类以 `org.springframework.aop.framework.Advised` 为旗帜，承载生成代理对象所需要的必要信息，比如相关目标类、Advice、Advisor 等。

ProxyConfig 其实就是一个普通的 JavaBean，它定义了5个 boolean 属性，分别控制在生成代理对象的时候，应该采取哪些行为措施：

1. **proxyTargetClass（默认false）**设置为 true 则会使用 CGLIB 对目标对象进行代理。
2. **optimize（默认false）**用于告知代理对象是否需要才去进一步的优化措施，如代理对象生成之后，即使为其添加或者移除了相应的 Advice，代理对象也可以忽略这种变动。
3. **opaque（默认false）**用于控制生成的代理对象是否可以强制转型为 Advised，默认是 false，表示任何生成的代理对象都可强制转型为 Advised，我们可以通过 Advised 查询代理对象的一些状态。
4. **exposeProxy（默认false）**是否将当前代理对象绑定到 ThreadLocal。出于性能考虑，默认不绑定。如果目标对象需要访问当前代理对象，可以通过 `AopContext.currentProxy()` 取得。
5. **frozen（默认false）**是否允许更改针对代理对象生成的各项信息配置，默认不允许，比如， ProxyFactory 设置完毕，并且 frozen 设置为 true，则不能对 Advice 进行任何变动，这样可以优化代理对象生成的性能。

要生成代理对象，只有 ProxyConfig 提供的控制信息还不够，我们还需要生成代理对象的一些具体信息，比如，要针对哪些目标类生成代理对象，要为代理对象加入何种横切逻辑等，这些信息都可以通过 `org.springframework.aop.framework.Advised` 设置或查询。默认情况下，Spring AOP 框架返回的代理对象都可以强制转型为 Advised，以查询代理对象的相关信息。

简单点说，我们可以使用 Advised 接口访问相应代理对象所持有的 Advisor，进行添加 Advisor、移除 Advisor 等相关动作。即使代理对象已经生成完毕，也可以对其进行这些操作。

AopProxy、AdvisedSupport 与 ProxyFactory 是什么关系呢？

![ProxyFactory继承层次图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ProxyFactory继承层次图.png)

ProxyFactory 集 AopProxy 和 AdvisedSupport 于一身，所以，可以通过 ProxyFactory 设置生成代理对象所需要的相关信息，也可以通过 ProxyFactory 取得最终生成的代理对象。前者是 AdvisedSupport 的职责，后者是 AopProxy 的职责。

为了重用相关逻辑，ProxyFactory 在实现的时候，将一些公用的逻辑抽取到了 `org.springframework.aop.framework.ProxyCreatorSupport` 中，它自身就继承了 AdvisedSupport，所以，生成代理对象的必要信息从其自身就可以获取到。为了简化子类生成不同类型的 AopProxy 的工作，ProxyCreatorSupport 内部持有一个 AopProxyFactory 实例，默认采用的是 DefaultAopProxyFactory，也可以通过构造方法或者是 setter 设置自定义的实现。

## ProxyCreatorSupport

ProxyFactory 只是 Spring AOP 中最基本的织入器实现，它还有另外的几个“兄弟”：

![ProxyFactory的“兄弟”](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ProxyFactory的“兄弟”.png)

## ProxyFactoryBean 介绍

在 IoC 容器中，使用 `org.springframework.aop.framework.ProxyFactoryBean` 作为织入器，它的使用与 ProxyFactory 无太大差别。ProxyFactoryBean 本质上是一个用来生产 Proxy 的 FactoryBean。如果容器中的某个对象持有某个 FactoryBean 的引用，它取得的不是 FactoryBean 本身，而是 FactoryBean 的 getObject 方法所返回的对象。所以，如果容器中某个对象依赖于 ProxyFactoryBean，那么它将会使用到 ProxyFactoryBean 的 getObject 方法所返回的代理对象。

要让 ProxyFactoryBean 的 getobject 方法返回相应目标对象类的代理对象其实很简单。因为 ProxyFactoryBean 继承了与 ProxyFactory 共有的父类 ProxyCreatorSupport，而ProxyCreatorSupport 基本上已经把要做的事情（如设置目标对象配置其他部件生成对应的 AopProxy 等）全部完成了。我们只需在 ProxyFactoryBean 的 getobject 方法中通过父类的createAopProxy 方法取得相应的 AopProxy，然后 `return AopProxy.getProxy()` 即可。

## ProxyFactoryBean 使用

ProxyFactoryBean 在继承了父类 ProxyCreatorSupport 的所有配置之外，还添加了几个自己独有的，如下所示：

- **proxyInterfaces：**如果要采用基于接口的代理方式，可以通过改属性配置相应的接口类型。实际上，它就是调用了 `AdvisedSupport::setInterfaces` 方法，二者等价，只是使用风格不同。另外，如果目标对象实现了某个或多个接口，即使不指定要代理的接口类型，ProxyFactoryBean 也可以自动检测到目标对象所实现的接口，并对其进行基于接口的代理。
- **singleton：**因为 ProxyFactoryBean 本质上是一个 FactoryBean，所以我们可以通过 singleton 属性，指定每次 getObject 调用是返回同一个代理对象，还是返回一个新的。通常情况下是返回同一个代理对象，即 singleton 为 true。只有在需要返回有状态的代理对象的情况下，才设置为 false，比如使用 Introduction 的场合。
- **interceptorNames：**可以通过改属性，一次性指定多个将要织入到目标对象的 Advice、Interceptor 以及 Advisor。可以在指定的 interceptorNames 某个元素名称之后添加 “*” 通配符，可以让 ProxyFactoryBean 在容器中搜寻符合条件的所有 Advisor 并应用到目标对象。

**Introduction 的织入**

引入一个 ICounter 接口定义以及一个简单实现类，然后将这个接口的行为和状态添加到 ITask 相应实现类中。

```java
public interface ICounter {
    void resetCounter();
    int getCounter();
}

@Component
@Scope("prototype")
public class CounterImpl implements ICounter {
    private int count;

    @Override
    public int getCounter() {
        return ++count;
    }

    @Override
    public void resetCounter() {
        count = 0;
    }
}
```

织入过程如下所示：

```java
@Bean
DelegatePerTargetObjectIntroductionInterceptor interceptor() {
    return new DelegatePerTargetObjectIntroductionInterceptor(CounterImpl.class, ICounter.class);
}

@Bean
@Scope("prototype")
ProxyFactoryBean proxyFactoryBean() throws ClassNotFoundException {
    final ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
    proxyFactoryBean.setTargetName("Task");
    proxyFactoryBean.setProxyInterfaces(new Class[]{ICounter.class, ITask.class});
    proxyFactoryBean.setInterceptorNames("interceptor");
    return proxyFactoryBean;
}
```

我们将目标对象的 Bean 定义、ProxyFactoryBean 的 Bean 定义，以及相应 IntroductionInterceptor 的 Bean 定义的 scope，全部声明为 prototype，也就是`singleton="false"`，并且，这种情况下，我们使用的是“taskName”而不是“task”来指定目标对象。这样才能保证每次取得的代理对象都持有各自独有的状态和行为。如下是调用执行的代码示例：

```java
@SpringBootApplication
public class SpringDemoApplication {

    public static void main(String[] args) {
        final ApplicationContext ac = SpringApplication.run(SpringDemoApplication.class, args);
        final ICounter counter1 = (ICounter) ac.getBean("proxyFactoryBean");
        final ICounter counter2 = (ICounter) ac.getBean("proxyFactoryBean");

        System.out.print(counter1.getCounter() + "  ");
        System.out.print(counter1.getCounter() + "  ");
        System.out.print(counter2.getCounter() + "  ");
    }

    // 输出：
    // 1  2  1
}
```

## ProxyFactoryBean 自动代理

要使用自动代理，需要以 Spring 的 IoC 容器为依托。Spring AOP 的自动代理的实现建立在 IoC 容器的 BeanPostProcessor 概念之上。通过 BeanPostProcessor，我们可以在遍历容器中所有 Bean 的基础上，对遍历到的 Bean 进行一些操作，有了这个前提条件，实现自动代理就很容易了。

只要提供一个 BeanPostProcessor，然后在这个 BeanPostProcessor 内部实现这样的逻辑，即当对象实例化的时候，为其生成代理对象并返回，而不是实例化后的目标对象本身，从而达到代理对象自动生成的目的。

Spring AOP 在 `org.springframework.aop.framework.autoproxy` 包中提供了两个常用的 AotoProxyCreator：BeanNameAutoProxyCreator 和 DefaultAdvisorAutoProxyCreator。

# TargetSource

TargetSource 的作用就好像是为目标对象在外面加了一个壳，就像是目标对象的容器。本来当每个针对目标对象的方法调用经历了层层拦截而到达调用链的终点的时候，就该去调用目标对象上定义的方法了。但 Spring AOP 在这里做了点手脚，它不是直接调用这个目标对象上的方法，而是通过 “插足于” 调用链与实际目标对象之间的某个 TargetSource 来取得具体的目标对象，然后再从 TargetSource 中取得的目标对象上的相应方法。

通常都是使用 setTarget 或者是 setTargetName 等方法来设置目标对象，但也还可以通过 setTargetSource 方法来指定目标对象。不论使用哪种方式，Spring AOP 框架内部都会通过一个 TargetSource 实现类对这个设置的目标对象进行封装。

TargetSource 最主要的特性就是，每次的方法调用都会触发 TargetSource 的 getTarget 方法，getTarget 方法将返回一个具体的目标对象。接口定义如下：

```java
public interface TargetSource extends TargetClassAware {
	Class<?> getTargetClass();
	boolean isStatic();
	Object getTarget() throws Exception;
	void releaseTarget(Object target) throws Exception;
}
```

- **getTargetClass：**返回目标对象类型；
- **isStatic：**用于表明是否要返回同一个目标对象实例，SingletonTargetSource 的这个方法肯定返回 true，其他的实现根据情况，通常返回 false；
- **getTarget：**核心方法，决定返回哪个目标对象实例；
- **releaseTarget：**释放当前调用的目标对象。但是否需要释放，完全是由实现的需要决定的，大部分时候，该方法可以只做空实现。

常见的实现类如下：

## SingletonTargetSource

`org.springframework.aop.target.SingletonTargetSource` 是使用的最多的 TargetSource 实现，通过名字都可以看出来它的实现很简单，就是内部只持有一个目标对象，当每次方法调用到达时，SingletonTargetSource 就会返回这同一个目标对象。

## PrototypeTargetSource

如果为 ProxyFactory 或者 ProxyFactoryBean 设置一个 PrototypeTargetSource 类型  TargetSource，那么每次方法调用到达调用链终点，并即将调用目标对象上的方法的时候，PrototypeTargetSource 都会返回一个新的目标对象实例供调用。但要注意目标对象的 scope 必须为 prototype。

## HotSwappableTargetSource

使用 HotSwappableTargetSource 封装目标对象，可以让我们在应用程序运行的时候，根据某种特定条件，动态地替换目标对象类的具体实现，比如，IService 有多个实现类，如果程序启动之后，默认的 IService 实现类出现了问题，我们可以马上切换到 IService 的另一个实现上，而所有这些对于调用者来说都是透明的。使用起来像这样：

```java
@Bean
protected HotSwappableTargetSource hotSwappableTargetSource() {
    return new HotSwappableTargetSource(new Task());
}

@Bean
protected ProxyFactoryBean proxy() {
    final ProxyFactoryBean proxy = new ProxyFactoryBean();
    proxy.setTargetSource(hotSwappableTargetSource());
    return proxy;
}

public static void main(String[] args) throws Exception {
    final ApplicationContext ac = SpringApplication.run(SpringDemoApplication.class, args);
    final Object proxy = ac.getBean("proxy");
    final HotSwappableTargetSource hotSwappableTargetSource = (HotSwappableTargetSource) ac.getBean("hotSwappableTargetSource");

    final ITask initTarget = (ITask) ((Advised) proxy).getTargetSource().getTarget();    
    final ITask oldTarget = (ITask) hotSwappableTargetSource.swap(new Task());
    final ITask newTarget = (ITask) ((Advised) proxy).getTargetSource().getTarget();

    assert initTarget == oldTarget;
    assert initTarget != newTarget;
}
```

> 使用场景：在有两个数据库双击热备的情况下，如果一个数据库挂掉，则将程序迅速地切换到另一个数据库。可以使用 ThrowsAdvice 对数据库相关异常进行捕获，在捕捉到必要的切换信息后，就调用 HotSwappableTargetSource 的 swap 方法使用新的数据源替换旧的数据源。

## CommonsPoolTargetSource

通过 CommonsPoolTargetSource 我们可以提供一个目标对象的对象池，然后让某个 TargetSource 实现每次都从这个目标对象池中去取得目标对象，就好像数据库连接池中的哪些 Connection 一样。CommonsPoolTargetSource 使用现有的 Jakarta Commons Pool 提供对象池支持。CommonsPoolTargetSource 使用起来跟 PrototypeTargetSource 没什么太大差别，也需要目标对象的 scope 必须为 prototype。

CommonsPoolTargetSource 还有许多控制对象池的可配置属性，比如对象池的大小、初始对象数量等。如果不能使用 Jakarta Commons Pool 对象池，那么也可以通过扩展 `org.springframework.aop.target.AbstractPoolingTargetSource`，实现相应的提供对象池化功能的 TargetSource。

## ThreadLocalTargetSource

通过 `org.springframework.aop.target.ThreadLocalTargetSource`，可以为不同的线程调用提供不同的目标对象，它可以保证各自线程上对目标对象的调用，可以被分配到当前线程对应的那个目标实现类实例上。同样，它的目标对象的 Bean 定义的 scope 也必须是 prototype。