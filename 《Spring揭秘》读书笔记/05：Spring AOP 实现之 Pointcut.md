<h1>Spring AOP 实现之 Pointcut</h1>

> 《Spring揭秘》读书笔记

- [ClassFilter](#classfilter)
- [MethodMatcher](#methodmatcher)
  - [NameMatchMethodPointcut](#namematchmethodpointcut)
  - [JdkRegexpMethodPointcut和Perl5RegexpMethodPointcut](#jdkregexpmethodpointcut和perl5regexpmethodpointcut)
  - [AnnotationMatchingPointcut](#annotationmatchingpointcut)
  - [ComposablePointcut](#composablepointcut)
  - [ControlFlowPointcut](#controlflowpointcut)
  - [自定义 StaticMethodMatcherPointcut](#自定义-staticmethodmatcherpointcut)
  - [自定义 DynamicMethodMatcherPointcut](#自定义-dynamicmethodmatcherpointcut)

**Joinpoint：** Spring AOP 中，仅支持方法执行类型的 Joinpoint。也就是仅能在方法内部执行开始时点进行横切逻辑织入。

Spring 中以接口定义 `org.springframework.aop.Pintcut` 作为其 AOP 框架中所有 Pointcut 的最顶层抽象，该接口定义了两个方法用来帮助捕捉系统中的相应 Joinpoint，并提供了一个 TruePointcut 类型实例。如果 Pointcut 类型为 TruePointcut ，默认会对系统中的所有对象，以及对象上所有被支持的 Joinpoint 进行匹配。`org.springframeworkaop.Pointcut` 接口定义如下:

```java
public interface Pointcut {
    ClassFilter getClassFilter();
    MethodMatcher getMethodMatcher();
    Pointcut TRUE = TruePointcut.INSTANCE;
}
```

# ClassFilter

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

# MethodMatcher

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
- **返回 false：** 表示不会考虑具体 Joinpoint 的方法参数，这种类型的 MethodMatcher 称为 `StaticMethodMatcher`。因为不用每次都检查参数，所以只有 `boolean matches(Method method, Class<?> targetClass);` 会被执行，那么对于同样类型的方法匹配结果，就可以在框架内部缓存以提高性能。
- **返回 true：** 表示该 MethodMatcher 将会每次都对方法调用的参数进行匹配检查，这种类型的 MethodMatcher 称之为 `DynamicMethodMatcher`，匹配效率相对于 `StaticMethodMatcher` 来说要差一些，最好避免使用。当 isRuntime 返回 true 后，并且当 `boolean matches(Method method, Class<?> targetClass);` 方法也返回 true 的时候，三个参数的 matches 方法才会被执行，以进一步检查匹配条件。如果两个参数的 matches 方法返回 false，那么不管 isRuntime 方法返回 true 还是 false，都不会执行三个参数 matches 方法。

各个 Pointcut 类型之间的关系图：

![Pointcut类型之间的关系图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Pointcut类型之间的关系图.png)

## NameMatchMethodPointcut

这是最简单的 Pointcut 实现，可以根据自身指定的一组方法名称与 Joinpoint 处的方法名称进行匹配。除了可以指定方法名，还可以使用 “*” 通配符，实现简单的模糊匹配。

但是，NameMatchMethodPointcut 无法对重载的方法名进行匹配，因为它仅对方法名进行匹配，不会考虑参数相关信息，而且也没有提供可以指定参数匹配信息的途径。

## JdkRegexpMethodPointcut和Perl5RegexpMethodPointcut

通过正则表达式匹配方法名。

## AnnotationMatchingPointcut

根据目标对象中是否存在指定类型的注解来匹配 Joinpoint，要使用该类型的 Pointcut，首先需要声明相应的注解。

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

## ComposablePointcut

Pointcut 通常还提供逻辑运算功能，而 ComposablePointcut 就是 Spring AOP 提供的可以进行 Pointcut 逻辑运算的 Pointcut 实现。像这样：

```java
ComposablePointcut pointcut1 = new ComposablePointcut(classFilter1, methodMatcher1);
ComposablePointcut pointcut2 = ...

ComposablePointcut unitedPoincut= pointcut1.union(pointcut2);
ComposablePointcut intersectionPointcut = pointcut1.intersection(unitedPoincut);

assertEquals(pointcut1,intersectionPointcut);
```

也能通过 `org.springframework.aop.support.Pointcuts` 工具类来实现：

```java
Pointcut pointcut1 = ...
Pointcut pointcut2 = ...

Pointcut unitedPoincut          = Pointcuts.union(pointcut1, pointcut2);
Pointcut intersectionPointcut   = Pointcuts.intersection(pointcutl, pointcut2);
```

## ControlFlowPointcut

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

如果使用之前的任何 Pointcut 实现，我们只能指定在 Targetobject 的 method1 方法每次执行的时候，都织入相应横切逻辑。也就是说，一旦通过 Pointcut 指定 method1 处为 Joinpoint，那么对该方法的执行进行拦截是必定的，不管 method1 是被谁调用。而通过 ControlFlowPointcut，我们可以指定，只有当 Targetobject 的 method1 方法在 TargetCaller 类所声明的方法中被调用的时候，才对 method1 方法进行拦截，其他地方调用 method1 的话，不对 method1 进行拦截。

ControlFlowPointcut 使用起来大概像这样：

```java
ControlFlowPointcut pointcut = new ControlFlowPointcut(TargetCaller.class);
Advice advice = ... ;

TargetObject target = new TargetObject();
TargetObject targetProxy = weaver.weave(advice).to(target).accordingTo(pointcut);

// advice 的逻辑在这里 “不会” 被触发执行
targetProxy.method1();

// advice 的逻辑在这里 “会” 被触发执行
// 因为 TargetCaller 的 callMethod() 将调用 method1()
// 正像 ControlFlowPointcut(TargetCaller.class) 所指定的那样
TargetCaller caller = new TargetCaller();
caller.setTarget(targetProxy);
caller.callMethod();
```

当织入器按照 Pointcut 的规定，将 Advice 织入到目标对象之后，从任何其他地方调用 method1，是不会触发 Advice 所包含的横切逻辑执行的。只有在 ControlFlowPointcut 规定的类内部调用目标对象的 method1，才会触发 Advice 中横切逻辑的执行。

> **注意：** 示例代码中 weaver 是虚构概念，不是 Spring 织入器的实现。

如果在 ControlFlowPointcut 的构造方法中单独指定 Class 类型的参数，那么 ControlFlowPointcut 将尝试匹配指定的 Class 中声明的所有方法，跟目标对象的 Joinpoint 处的方法流程组合。如果还想指定调用类的某个具体方法，可以在构造 ControlFlowPointcut 的时候，传入第二个参数，即调用方法的名称：

```java
ControlFlowPointcut pointcut = new ControlFlowPointcut(TargetCaller.class, "callMethod");
```

## 自定义 StaticMethodMatcherPointcut

> 继承抽象类 StaticMethodMatcher 即可。

StaticMethodMatcher 根据自身语意，为其子类提供了如下几个方面的默认实现。

- 默认所有 StaticMethodMatcher 的子类的 ClassFilter 均为 `ClassFilter.TRUE` 即忽略类的类型匹配。如果具体子类需要对目标对象的类型做进一步限制，可以通过 `public void setclassFilter(ClassFilter classFilter)` 方法设置相应的 ClassFilter 实现。
- 因为是 staticMethodMatcher，所以，其 MethodMatcher 的 isRuntime 方法返回 false，同时三个参数的 matches 方法抛出 `UnsupportedOperationException` 异常，以表示该方法不应该被调用到。

最终我们需要做的就是实现两个参数的 matches 方法了

## 自定义 DynamicMethodMatcherPointcut

> 继承抽象类 DynamicMethodMatcher 即可。

DynamicMethodMatcherPointcut 也为其子类提供了部分便利。

- getclassFilter() 方法返回 `ClassFilter.TRUE`，如果需要对特定的目标对象类型进行限定子类只要覆写这个方法即可。
- 对应的 MethodMatcher 的 isRuntime 总是返回 true，同时，DynamicMethodMatcher 提供了两个参数的 matches 方法的实现，默认直接返回 true。

要实现自定义 DynamicMethodMatcherPointcut，通常情况下，我们只需要实现三个参数的 matches 方法逻辑即可。但如果愿意，我们也可以覆写一下两个参数的 matches 方法，这样，不用每次都得到三个参数的 matches 方法执行的时候才检查所有的条件。
