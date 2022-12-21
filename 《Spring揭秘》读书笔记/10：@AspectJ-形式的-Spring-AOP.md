<h1>@AspectJ 形式的 Spring AOP</h1>

> 《Spring揭秘》读书笔记

- [@AspectJ 形式的AOP](#aspectj-形式的aop)
- [@AspectJ 形式：Pointcut 的声明](#aspectj-形式pointcut-的声明)
  - [execution](#execution)
  - [within](#within)
  - [this 和 target](#this-和-target)
  - [args](#args)
  - [@within](#within-1)
  - [@target](#target)
  - [@args](#args-1)
  - [@annotation](#annotation)
- [@AspectJ 形式：Pointcut 的实现](#aspectj-形式pointcut-的实现)
- [@AspectJ 形式：Advice](#aspectj-形式advice)
  - [@Before](#before)
  - [@AfterThrowing](#afterthrowing)
  - [@AfterReturning](#afterreturning)
  - [@After](#after)
  - [@Around](#around)
  - [@DeclareParents](#declareparents)
  - [Advice 的执行顺序](#advice-的执行顺序)
- [@AspectJ 形式：Aspect 的实例化模式](#aspectj-形式aspect-的实例化模式)

Spring 框架 2.0 版本发布之后，Spring AOP 增加了新的特性，或者说增加了新的使用方式。2.0之后的 Spring AOP 集成了 AspectJ，但底层的各种概念的实现以及织入方式，依然使用的是 Spring 1.x 原先的实现体系。

@AspectJ 代表一种定义 Aspect 的风格，它让我们能够以 POJO 的形式定义 Aspect，没有其他接口定义限制。唯一需要的，就是使用相应的注解标注这些 Aspect 定义的 POJO 类。之后，Spring AOP 会根据标注的注解搜索这些 Aspect 定义类，然后将其织入系统。

# @AspectJ 形式的AOP

看看 @AspectJ 形式的 AOP 应该怎么使用，定义一个用于性能监控的 AOP：

```java
@Aspect
public class PerformanceTraceAspect {

    @Pointcut("execution(public void *.method1()) || execution(public void *.method2())")
    public void pointcutName() { }

    @Around("pointcutName()")
    public Object performanceTrace(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("start_time : " + System.currentTimeMillis());
        try {
            return joinPoint.proceed();
        } finally {
            System.out.println("end_time : " + System.currentTimeMillis());
        }
    }
}
```

如下是目标类定义：

```java
public class Foo {
    public void method1() {
        System.out.println("method1 execution.");
    }

    public void method2() {
        System.out.println("method2 execution.");
    }
}
```

可以通过编码的方式织入，像这样：

```java
AspectJProxyFactory weaver = new AspectJProxyFactory();
weaver.setTarget(new Foo());
weaver.addAspect(PerformanceTraceAspect.class);
((Foo) weaver.getProxy()).method1();
((Foo) weaver.getProxy()).method2();
```

> AspectJProxyFactory 的使用与 ProxyFactory 没有多大区别，如果愿意，也完全可以把 AspectJProxyFactory 当作 ProxyFactory 来用。

也可以通过自动代理织入，如果你使用了 Spring Boot 来构建 Spring 项目，那你只需要在启动类上加这个注解 `@EnableAspectJAutoProxy` 来开启自动代理，它会使用 `org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator` 类来自动搜集 IoC 容器中注册的 Aspect，并应用到 Pointcut 定义的各个目标对象上。此时，如果再通过容器获取 target 对象的话，会发现它已经是被代理过的了。像这样：

```java
@SpringBootApplication
@EnableAspectJAutoProxy
public class SpringDemoApplication {

    public static void main(String[] args) throws Exception {
        final ApplicationContext ac = SpringApplication.run(SpringDemoApplication.class, args);

        ((Foo) ac.getBean("foo")).method1();
        ((Foo) ac.getBean("foo")).method2();
    }
}
```

对于执行结果，你不妨自己试试看。

# @AspectJ 形式：Pointcut 的声明

@AspectJ 形式的 Pointcut 声明，依附在 @Aspect 所标注的 Aspect 定义类之内，通过使用 `org.aspectj.lang.annotation.Pointcut` 这个注解，指定 AspectJ 形式的 Pointcut 表达式之后，将这个指定了相应表达式的注解标注到 Aspect 定义类的某个方法上即可。比如刚刚 PerformanceTraceAspect 的 pointcutName 方法上的 `@Pointcut` 注解一样。

**@AspectJ 形式的 Pointcut 声明包含如下两个部分：**

1. **Pointcut Expression。** Pointcut Expression 的载体为 @Pointcut，该注解是方法级别的注解，所以 Pointcut Expression 不能脱离某个方法单独声明。Pointcut Expression 是真正规定 Pointcut 匹配规则的地方，可以通过 @Pointcut 直接指定 AspectJ 形式的 Pointcut 表达式，由两部分组成，分别是标识符和匹配模式，将在下文详细说明。
2. **Pointcut Signature。** Pointcut Signature 在这里具体化为一个方法定义，它是 Pointcut Expression 的载体。Pointcut Signature 所在的方法定义，除了返回类型必须是 void 之外，没有其他限制。方法修饰符所起的作用与 Java 语言中语义相同。可以将 Pointcut Signature 作为相应 Pointcut Expression 标识符，在 Pointcut Expression 的定义中取代重复的 Pointcut 表达式定义，像这样：

```java
@Aspect
public class PointcutExpressionDemo {
    @Pointcut("execution(void method1())")
    public void method1Execution() { };

    @Pointcut("method1Execution()")
    public void stillMethod1Execution() { };
}
```

AspectJ 的 Pointcut 表达式支持通过 `&&`、`||` 以及 `!` 逻运算符，进行 Pontcut 表达式之间的逻辑运算运算符可以应用于具体的 Pointcut 表达式，以及相应的 Pointcut Signature。

---

以下是 @AspectJ 形式 Pointcut 表达式的标志符

## execution

execution 可以帮助我们匹配拥有指定方法前面的 Joinpoint，使用 execution 标志符的 Pointcut 表达式的规定格式如下：

```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern)
```

其中，方法的返回类型、方法名以及参数部分的匹配模式是必须指定的，其他部分的匹配模式可以省略。PerformanceTraceAspect.pointcutName 方法使用的就是 execution 标识符。

还可以在 execution 表达式中使用两种通配符，即 `*` 和 `..`

- `*`：可以用于任何部分的匹配模式中，可以匹配相邻的多个字符，即一整个单词。
- `..`：只能在 `declaring-type-pattern` 和 `param-pattern` 的位置使用。
  - 使用在 `declaring-type-pattern`：可以指定多个层次的类型声明，如下：
    ```
    // 只能指定到 cn.spring21 这层下的所有类型
    execution(void cn.spring21.*.doSomething(*))
    // 可以匹配到 cn.spring21 包下的所有类型，以及 cn.spring21 下层包下声明的所有类型
    execution(void cn.spring21..*.doSomething(*))
    ```
  - 使用在 `param-pattern`：表示该方法可以有0到多个参数，参数类型不限，如下：
    ```
    execution(void *.doSomething(..))
    ```

execution 使用示例：

```
// 匹配两个参数的 doSomething 方法，第一个参数为 String 类型，第二个参数类型不限
execution(void doSomething(string, *))

// 匹配拥有多个参数的 doSomething 方法，之前几个参数类型不限，但最后一个参数类型必须为 String
execution(void doSomething(.., String))

// 匹配拥有多个参数的 doSomething 方法，第一个参数类型不限，第二个参数类型为 String，其它剩余参数数量，类型均不限
execution(void doSomething(*,String,..))
```

## within

within 标志符只接收类型声明，它将会匹配指定类型下的所有 Joinpoint。不过，因为 Spring AOP 只支持方法级别的 Joinpoint，所以，在我们为 within 指定某个类后，它将匹配指定类所声明的所有方法执行。假设我们声明一个使用 within 标志符的 Pointcut 像这样：`within(cool.wrp.springdemo.aop.Foo)`。

那么，该 Pointcut 表达式在 Spring AOP 中将会匹配 Foo 类中的所有方法声明。另外，我们也可以通过 `*` 和 `..` 通配符来扩展匹配的类型范围，如下所示：

```
// 匹配 cool.wrp.springdemo.aop 包下所有类型内部的方法级别的 Joinpoint
within(cool.wrp.springdemo.aop.*)

// 匹配 cool.wrp.springdemo.aop 以及其子包下所有类型的内部方法级别的 Joinpoint
within(cool.wrp.springdemo.aop..*)
```

## this 和 target

在 AspectJ 中，this 指代调用方法一方所在的对象（caller），target 指代被调用方法所在的对象（callee），这样通常可以同时使用这两个标志符限定方法的调用关系。比如，如果 Object1、Object2 都会调用 Object3 的某个方法，那么 Pointcut 表达式定义 `this(Object1) && target(Object3)`，只会当 Object1 调用 Object3 上的方法的时候才会匹配，而 Object2 调用 Object3 上的方法则不会被匹配。

Spring AOP 中的 this 和 target 标志符语义，有别于 AspectJ 中两个标志符的原始语义。Spring AOP 中 this **指代的是目标对象的代理对象**，而由于 Spring AOP 只支持方法执行类型的 Joinpoint，所以 Spring AOP 中的目标对象的语义刚好与 target 一致，target 就**指代的就还是目标对象**。

**至于为什么，这块书上没说，但我是这么理解的：**

因为 AOP 实际上是一个规范，而 Spring AOP 是这个规范的一个具体实现，所以一些 AOP 中的内容可能由于 Java 的特性没办法完美的实现出来，比如 AOP 就没规定必须使用代理实现吧，这里的 this 和 target 也是如此。

**this 为什么有别于 AOP 的 this，我想应该是这样：**

AOP 中 this 的语义很明显，就是指代调用方（caller）对象，但由于 Java 是静态语言，caller 对象被编写出来之后，Java 没有更改 caller 的能力，所以在 AOP 的实现上 Java 采用了曲线救国的方式，也就是代理。那么如果 this 的语义还是指代 caller 对象的话，Java 没有能力为 caller 做横切逻辑的织入。所以在这里 this 的语义**指代的是目标对象的代理对象**。

实际上，从代理模式来看，代理对象通常跟目标对象的类型是相同的，因为目标对象与它的代理对象实现同一个接口。即使使用 CGLIB 的方式，目标对象的代理对象属于目标对象的子类，通过单独使用 this 或者 target 指定类型，起到的限定作用其实是差不多的。假设我们现在有如下的对象定义：

```java
public interface ProxyInterface {
    // ...
}

public class TargetFoo implements ProxyInterface {
    // ...
}
```

不论使用基于接口的代理方式，还是基于类的代理方式 `this(ProxyInterface)` 和 `target(ProxyInterface)` 这两个表达式所起的效果实际上是差不多的。因为 TargetFoo 作为目标对象实现了 ProxyInterface。对于基于接口的代理来说，它的代理对象同样实现了这个接口。对于基于类的代理来说，因为目标对象的代理对象是继承了目标对象，自然也继承了目标对象实现的接口。所以这两个 Pointcut 定义起得作用差不多。

但如果指定到具体类型，像这样：`this(TargetFoo)` 和 `target(TargetFoo)`。这时，如果还是基于类的代理来说，这两个 Pointcut 表达式限定的语义还是基本相同的。但基于接口的代理就有区别了，`target(TargetFoo)` 还是可以匹配到目标对象中所有的 Joinpoint，因为目标对象确实是 TargetFoo 类型，而 `this(TargetFoo)` 则不可以，因为 this 匹配的是目标对象的代理，这里只能匹配到代理对象，而匹配不到 TargetFoo 对象。

通常，this 和 target 标志符都是在 Pointcut 表达式中与其他标志符结合使用，以进一步加强匹配的限定规则。对于 Introduction 来说，代理对象所实现的接口数量通常比目标对象多，可以通过同时使用 this 和 target 进一步限定匹配规则，比如：

`this(IntroductionInterface) && target(TargetObjectType)`

Introduction 为目标对象动态添加了新的接口，但是新的接口添加到了目标对象的代理对象上。那么以上表达式就可以匹配到 Introduction 代理对象（TargetObjectType）上的特定接口（IntroductionInterface）中的方法。

## args

args 标志符可以帮助我们捕捉拥有指定参数类型、指定参数数量的方法级 Joinpoint，而不管该方法在什么类型中被声明。如果我们声明了这样的 Pointcut 表达式：`args(User)`，那么，只要参数是单个的 User 对象都会被匹配到，如下的方法签名都将被该 Pointcut 所匹配：

```java
public class Foo {
    public boolean login(User user) { };
}

public class Bar {
    public boolean isLogin(User user) { };
}
```

与使用 execution 标志符可以直接明确指定方法参数类型不同，args 标志符会在运行期间动态检查参数的类型，即使是这样的方法签名：`public boolean login(Object user)`，只要传入的是 User 类型的实例，那么使用 args 标志符的 Pointcut 表达式依然可以捕捉到该 Joinpoint，但 execution 则捕捉不到了。args 同样可以使用 `*` 和 `..` 通配符。

## @within

如果使用 @within 指定了某种类型的注解，那么，只要对象标注了该类型的注解，使用了 @within 标志符的 Pointcut 表达式将匹配该对象内部所有的 Joinpoint。对于 Spring AOP 来说，当然是对象内部声明的所有方法级 Joinpoint。假设我们声明注解如下：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface AnyJoinpointAnnotation {
}
```

该注解拥有类声明如下：

```java
@AnyJoinpointAnnotation
public class Foo {
    public void method1() { }
    public void method2() { }
}
```

这样，当使用 @within 标志符声明 Pointcut 表达式时 `@within(AnyJoinpointAnnotation)`，Foo 类中的 method1、method2 等方法将全部被该 Pointcut 表达式所匹配。

## @target

使用方式与 @within 相同，在 Spring AOP 中，@within 和 @target 没有太大的区别，只不过 @within 属于静态匹配，而 @target 则是在运行时点动态匹配 Joinpoint。

## @args

使用 @args 标志符的 Pointcut 表达式将会尝试检查当前方法级的 Joinpoint 的方法参数类型。如果该次调用传入的参数类型拥有 @args 所指定的注解，当前 Joinpoint 将被匹配，否则将不会被匹配。

以 @within 部分 @AnyJoinpontAnnotation 为例，如果指定 Pointcut 表达式像这样：`@args(AnyJoinpointAnnotation) && execution(public void *.method(*))`。该注解拥有类声明如下：

```java
public interface IArgsProxy {
    @AnyJoinpointAnnotation
    class ArgsProxyImpl_1 implements IArgsProxy {
    }

    class ArgsProxyImpl_2 implements IArgsProxy {
    }
}
```

目标类 Foo 像这样：

```java
@Component
public class Foo {
    public void method(IArgsProxy proxy) {
        System.out.println("test");
    }
}
```

调用的时候，像这样：

```java
public static void main(String[] args) throws Exception {
    final ApplicationContext ac = SpringApplication.run(SpringDemoApplication.class, args);

    final Foo foo = ac.getBean("foo", Foo.class);
    foo.method(new IArgsProxy.ArgsProxyImpl_1()); // 被代理
    foo.method(new IArgsProxy.ArgsProxyImpl_2()); // 不被代理
}
```

@args 会尝试对系统中所有对象的每次方法执行的参数，都进行指定的注解的动态检查。只要参数类型标注有 @args 指定的注解类型，当前方法执行就将被匹配，至于说参数类型是什么，它并不是十分关心。

## @annotation

使用 @annotation 标志符的 Pointcut 表达式，将会尝试检查系统中所有对象的所有方法级别 Joinpoint。如果被检测的方法标注有 @annotation 标志符所指定的类型，那么当前方法所在的 Joinpoint 将被 Pointcut 表达式所匹配。

@annotation 标志符的应用场景也比较广泛，假设我们要对系统中的事务处理进行统一管理，可以通过团队内部规定的使用注解的方式管理事务。当开发人员希望对某个方法加事务控制的时候，只要使用相应的注解标注一下即可。

需要注意的是，@annotation 所接受的注解类型只应用于方法级别，即标注了 `@Target(ElementType.METHOD)` 的注解声明。

# @AspectJ 形式：Pointcut 的实现

实际上，@AspectJ 形式声明的所有 Pointcut 表达式，在 Spring AOP 内部都会通过解析，转化为具体的 Pointcut 对象。因为 Spring AOP 有自己的 Pointcut 定义结构，所以，@AspectJ 形式声明的这些 Pointcut 表达式，最终都会转化成一个专门面向 AspectJ 的 Pointcut 实现。

`org.springframework.aop.aspectj.AspectJExpressionPointcut` 代表 Spring AOP 中面向 AspectJ 的 Pointcut 具体实现。虽然它会使用 AspectJ 的相应支持，但仍然遵循 Spring AOP 的 Pointcut 定义，其在 Pointcut 中的地位如图所示：

![AspectJPointcut扩展的类图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/AspectJPointcut扩展的类图.png)

`ExpressionPointcut` 和 `AbstractExpressionPointcut` 主要是为了以后的扩展性。如果还有除 AspectJ 以外的 Pointcut 描述形式，可以在这个基础之上扩展。

在 AspectJProxyFactory 或者 AnnotationAwareAspectJAutoProxyCreator 通过反射获取了 Aspect 中的 @Pointcut 定义的 AspectJ 形式的 Pointcut 定义之后，在 Spring AOP 框架内部都会构造一个对应的 AspectJExpressionPointcut 对象实例。也就是说，AspectJExpressionPointcut 内部持有通过反射获得的 Pointcut 表达式。

AspectJExpressionPointcut 依然属于 Spring AOP 的 Pointcut 定义之一，所以，Spring AOP 框架内部处理 Pointcut 匹配的逻辑不需要改变，依然使用原来的匹配机制（通过 ClassFilter 和 MethodMatcher 进行具体 Joinpoint 匹配工作）。

不过 AspectJExpressionPointcut 在实现 ClassFilter 和 MethodMatcher 相应方法逻辑的时候，会委托 AspectJ 类库中的 PointcutParser 来对它所持有的 AspectJ 形式的 Pointcut 表达式进行解析。PointcutParser 解析后会返回一个 PointcutExpression 对象（依然是 AspectJ 类库中的类），之后，匹配与否就直接委托这个 PointcutExpression 对象进行处理了。

# @AspectJ 形式：Advice

@AspectJ 形式的 Advice 定义，实际上就是使用 @Aspect 标注的 Aspect 定义类中的普通方法。只不过，这些方法需要针对不同的 Advice 类型使用对应的注解进行标注。

可以用于标注对应 Advice 定义方法的注解包括：

- `@Before`、`@AfterReturning`、`@AfterThrowing`、`@After`、`@Around`
- `@DeclareParents`，用于标注 Introduction 类型的 Advice，但该注解标注的对象是`域(Field)`，而不是`方法(Method)`

各种 Advice 最终织入到什么位置，是由相应的 Pointcut 定义决定的。所以，我们需要为这些 Advice 注解指定对应的 Pointcut 定义，可以直接指定 @AspectJ 形式的 Pointcut 表达式，也可以指定单独声明的 @Pointcut 类型定义的 Pointcut Signature（不包括 @DeclareParents）

---

这些注解的使用大体上都跟 PerformanceTraceAspect 相似，以下列举一些不同之处。

## @Before

某些情况下，我们可能需要在 Advice 定义中访问 Joinpoint 处的方法参数，在 1.x 的 Spring AOP 中，我们可以通过 MethodBeforeAdvice 的 before 方法传入的 Object[] 数组，访问相应的方法参数。现在，我们有如下两种方式达到相同的目的：

1. `org.aspectj.lang.JoinPoint`，被 @Before 注解标注的方法可以将第一个参数声明为 JoinPoint 类型（JoinPoint 类型的参数必须在参数列表的第一位），通过 JoinPoint 我们可以借助它的 getArgs 方法，访问相应 Joinpoint 处方法的参数值。另外，getThis 方法可以获取当前代理对象；getTarget 方法可以取得当前目标对象；getSignature 方法可以取得当前 Joinpoint 处的方法签名。
2. **通过 args 标志符绑定。** args 标志符除了可以限定 Pointcut 定义之外，当 args 标志符接受的不是具体的对象类型，而是某个参数名称的时候，它会将这个参数名称对应的参数值绑定到 Advice 方法的调用。使用如下：

```java
@Before("execution(...) && args(name)")
public void beforeAdvice(String name) {
    // ...
}
```

注意：args 指定的参数名称必须与 Advice 定义所在方法的参数名称相同。在这里，args 指定的值和 beforeAdvice 方法的参数名都是 name。

以上如果 Advice 引用的是独立的 Pointcut 定义，可以这么写：

```java
@Pointcut("execution(...) && args(name)")
public void pointcutName(String name) { }

@Before("pointcutName(name)")
public void beforeAdvice(String name) {
    // ...
}
```

---

**由此扩展：**

1. 在 @AspectJ 形式的 Advice 声明中，除了 Around Advice 和 Introduction 之外，余下的 Advice 类型声明如果需要 `org.aspectj.lang.JoinPoint` 参数，必须将其声明在参数列表中的第一位。
2. 不只可以使用 args 标志符来绑定参数声明到方法，实际上，Pointcut 标志符中，除了 execution 标志符不会直接指定对象类型之外，其他像 this、target、@within、@target、@annotation、@args 等原本都是指定对象类型的。它们与 args 一样，在这样的场合下，如果它们指定的是参数名称，那么，所起的作用与 args 在这种参数绑定的场景中的作用是相似的，比如：

```java
@Before("execution(...) && args(name) && annotation(txInfo)")
public void beforeAdvice(String name, Transactional txInfo) {
    // 访问 name 参数
    // 通过 txInfo 参数访问事务控制相关信息
}
```

## @AfterThrowing

@AfterThrowing 有一个 throwing 属性，该属性可以限定 Advice 定义方法的参数名，并在方法带哦用的时候，将相应的异常绑定到具体的方法参数上，之后就可以在方法内部访问具体的异常信息，如下：

```java
@AfterThrowing(pointcut = "execution(...)", throwing = "e")
public void afterThrowingAdvice(RuntimeExeception e) {
    // ...
}
```

与 1.x 中的 ThrowsAdvice 可以同时针对不同的异常类型声明不同的方法进行处理一样，我们也可以在 Aspect 中针对多个不同的异常类型，声明不同的 AfterThrowingAdvice 的方法定义。

## @AfterReturning

某些时候需要访问方法的返回值，可以通过 @AfterReturning 的 returning 属性将返回值绑定到 After Returning Advice 定义所在的方法，像这样：

```java
@AfterReturning(pointcut = "execution(...)", returning = "retValue")
public void afterReturningAdvice(boolean retValue) {
    // ...
}
```

## @After

对于匹配的 Joinpoint 处的方法执行来说，不管该方法是正常执行返回，还是执行过程中抛出异常而非正常返回，都会触发其上的 After（Finally）Advice 执行，一般适用于释放某些资源的场景比如网络连接的释放、数据库连接资源的释放。使用起来没有什么特殊的地方，就不做代码展示了。

## @Around

被 @Around 注解标注的方法的第一个参数必须是 `org.aspectj.lang.ProceedingJoinPoint` 类型，且必须指定该参数。通常需要通过 `ProceedingJoinPoint::proceed()` 方法继续调用链的执行。除了直接调用 proceed 方法，还可以在调用 proceed 方法的时候，传入一个 Object[] 数组代表方法参数列表。如果想要修改方法调用参数值的时候，就可以使用带 Object[] 参数的 proceed 方法。

PerformanceTraceAspect 中使用的就是 @Advice。

## @DeclareParents

以 @AspectJ 形式声明的 Introduction，完全不同于之前提到过的任何 Advice，@DeclareParents 注解不是对 Aspect 中各 Advice 方法进行标注，而是对 Aspect 中的实例变量定义进行标注。

在 Spring 中，Introduction 的实现是通过将需要添加的新的行为逻辑，以新的接口定义增加到目标对象上。要以 @AspectJ 形式声明 Introduction，我们需要在 Aspect 中声明一个实例变量它的类型对应的就是新增加的接口类型，然后使用 @DeclareParents 对其进行标注。通过 @DeclareParents 指定新接口定义的实现类以及将要加诸其上的目标对象。使用起来像这样，为 Task 添加 ICounter 的接口实现：

```java
@Aspect
public class IntroductionAspect {
    @DeclareParents(
        value = "...Task", // 指定目标对象
        defaultImpl = CounterImpl.class // 新增接口的实现类
    )
    // 为目标对象新增的对象类型
    public ICounter counter;
}
```

指定目标对象的时候，可以使用通配符指定一批目标对象，假设 service 包下的所有类都想增加 ICounter 的行为，那么可以把 value 定义成这样：service.*。

注意：Introduction 属于 per-instance 类型的 Advice，所以目标对象的 scope 通常情况下应该设置成 prototype。

## Advice 的执行顺序

对于多个 Advice 来说，如果它们引用的 Pointcut 定义恰好匹配到同一个 Joinpoint 的时候，在这个 Joinpoint 上，这些 Advice 的执行顺序有以下规律：

1. 当这些 Advice 都声明在**同一个 Aspect 内**的时候，那么这些 Advice 的执行顺序，由它们在 Aspect 中的声明顺序来决定，最先声明的 Advice 拥有最高的优先级。对于 Before Advice 来说，拥有最高优先级的最先运行，而对于 AfterReturningAdvice，拥有最高优先级的则是最后运行。

```java
@Aspect
public class MultiAdvicesAspect {
    @Pointcut("execution(...)")
    public void executionPointcut();

    @Before("executionPointcut()")
    public void beforeOne() {
        System.out.println("before one");
    }

    @Before("executionPointcut()")
    public void beforeTwo() {
        System.out.println("before two");
    }

    @AfterReturning("executionPointcut()")
    public void afterReturningOne() {
        System.out.println("after returning one");
    }

    @AfterReturning("executionPointcut()")
    public void afterReturningTwo() {
        System.out.println("after returning two");
    }
}
```

如果将该 Aspect 织入目标对象，那么可以得到如下的输出结果：

```
before One
before Two
task executed
after returning two
after returning one
```

@Before 的输出很容易理解，尤其需要注意 @AfterReturning 的输出，afterReturningOne 方法先于afterReturningTwo 方法声明，拥有较高的优先级，但因为它属于 AfterReturning Advice，所以，要晚于 afterReturningTwo 方法的执行。

2. 当这些 Advice 声明都在**不同 Aspect 内**的时候，这时需要用到 Spring 的 `org.springframework.core.Ordered` 接口，只需要让相应的 Aspect 定义实现 Ordered 接口即可，否则，Advice 的执行顺序是无法确定的。Ordered.getorder() 方法返回的值越小 Aspect，其内部所声明的Advice 拥有的优先级越高。

注意：如果使用编程的方式来使用这些 Aspect，那么 Aspect 内的 Advice 执行顺序完全由添加到 AspectJProxyFactory 的顺序来决定了。

# @AspectJ 形式：Aspect 的实例化模式

Spring 2.0 之后的 AOP 仅支持 AspectJ 的三种实例化模式，即 singleton、perthis 和 pertarget。默认是 singleton 单例模式，也就是说，在容器中会实例化并持有每个 Aspect 定义的单一实例。

- perthis：为代理对象实例化各自的 Aspect 实例
- pertarget：为匹配的单独的目标对象实例化相应的 Aspect 实例

使用起来像这样：

```java
@Aspect("perthis(execution(...))") // 为指定的 pointcut 创建 perthis 实例化模式的 Aspect
public class InstanceModeAspect {
    // ...
}
```

不过，使用 perthis 或者 pertarget 指定了 Aspect 的实例化模式之后，将这些 Aspect 注册到容器时，不能为其 Bean 定义指定 singleton 的 scope，否则会出现异常。