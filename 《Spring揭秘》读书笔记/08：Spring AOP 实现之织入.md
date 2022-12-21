<h1>Spring AOP 实现之织入</h1>

> 《Spring揭秘》读书笔记

- [ProxyFactory 使用](#proxyfactory-使用)
  - [基于接口的代理](#基于接口的代理)
  - [基于类的代理](#基于类的代理)
  - [Introduction 的织入](#introduction-的织入)
- [ProxyFactory 实质](#proxyfactory-实质)
  - [AopProxyFactory](#aopproxyfactory)
  - [AdvisedSupport](#advisedsupport)
- [ProxyCreatorSupport](#proxycreatorsupport)
- [ProxyFactoryBean 介绍](#proxyfactorybean-介绍)
  - [ProxyFactoryBean 使用](#proxyfactorybean-使用)
  - [ProxyFactoryBean 自动代理](#proxyfactorybean-自动代理)

在 Spring AOP 中，使用类 `org.springframework.aop.framework.ProxyFactory` 作为织入器。ProxyFactory 并非 Spring AOP 中唯一可用的织入器，而是最基本的一个织入器实现。

Spring AOP 是基于代理模式的 AOP 实现，织入过程完成后，会返回织入了横切逻辑的目标对象的代理对象。使用 ProxyFactory 只需要指定如下两个最基本的东西：

- **要对其进行织入的目标对象。** 可以通过 ProxyFactory 的构造器或者 setter 注入。
- **将要应用到目标对象的 Aspect，也就是 Spring 里面的 Advisor。**
  - 对于 Inroduction 之外的 Advice 类型，ProxyFactory 内部就会为这些 Advice 构造相应的 Advisor，只不过在为它们构造的 Advisor 中使用的 Pointcut 为 `Pointcut.TRUE`，这些 Advice 将被应用到系统中所有可识别的 Joinpoint 处。
  - 如果要添加的 Advice 类型是 Inroduction 类型，**则会根据该 Inroduction 的具体类型进行区分：**
    - IntroductionInfo：因为它本身包含了必要的描述信息，框架内部会为其构造一个 DefaultIntroductionAdvisor；
    - DynamicIntroductionAdvice：框架内部将抛出 AopConfigException 异常（因为无法从DynamicIntroductionAdvice取得必要的目标对象信息）

# ProxyFactory 使用

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

## 基于接口的代理

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

简单点儿说，如果目标类实现了至少一个接口，不管我们有没有通过 ProxyFactory 的setInterfaces 方法明确指定要对特定的接口类型进行代理，只要不将 ProxyFactory 的 `optimize` 和 `proxyTargetclass` 两个属性的值设置为 true（这两个属性稍后将谈到），那么 ProxyFactory 都会按照面向接口进行代理。

## 基于类的代理

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

## Introduction 的织入

Introduction 型的 Advice 比较特殊，如下所述：

- Introduction 可以为已经存在的对象类型添加新的行为，只能应用于对象级别的拦截，而不是通常 Advice 的方法级别的拦截，所以，进行 Inroduction 的织入过程中，不需要指定 Pointcut，而是需要指定目标接口类型。
- Spring 的 Introduction 支持只能通过接口定义为当前对象添加新的行为，所以，我们需要在织入的时机，指定新织入的接口类型。

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

# ProxyFactory 实质

为 ProxyFactory 提供具体代理机制实现的接口：`org.springframework.aop.framework.AopProxy`

```java
public interface AopProxy {
	Object getProxy();
	Object getProxy(@Nullable ClassLoader classLoader);
}
```

Spring AOP 框架内使用 AopProxy 对使用的不同的代理实现机制进行了适度的抽象，针对不同的代理实现机制提供相应的 AopProxy 子类实现。目前，Spring AOP 框架内提供了针对 JDK 的动态代理和 CGLIB 两种机制的 AopProxy 实现。

![AopProxy相关结构图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/AopProxy相关结构图.png)

## AopProxyFactory

因为动态代理需要通过 InvocationHndler 提供调用拦截，所以 JdkDynamicAopProxy 同时实现了 InvocationHndler 接口。不同 AopProxy 实现的实例化过程采用工厂模式（确切地说是抽象工厂模式）进行封装，即通过 `org.springframework.aop.framework.AopProxyFactory` 进行。其定义如下：

```java
public interface AopProxyFactory {
    AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;
}
```

AopProxyFactory 根据传入的 AdvisedSupport 实例提供的相关信息，来决定生成什么类型的 AopProxy。不过具体的工作会转交给它的实现类：`org.springframework.aop.framework.DefaultAopProxyFactory`。它的工作逻辑很简单，就是判断是否采用 CGLIB 生成动态代理对象，满足直接生成，否则生成 JDK 动态代理对象。

## AdvisedSupport

AdvisedSupport 其实就是一个生成代理对象所需要的信息的载体，该类相关的类层次图如下：

![AdvisedSupport类层次图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/AdvisedSupport类层次图.png)

AdvisedSupport 所承载的信息可以划分为两类，一类以 `org.springframework.aop.framework.ProxyConfig` 为统领，记载生成代理对象的控制信息；一类以 `org.springframework.aop.framework.Advised` 为旗帜，承载生成代理对象所需要的必要信息，比如相关目标类、Advice、Advisor 等。

ProxyConfig 其实就是一个普通的 JavaBean，它定义了5个 boolean 属性，分别控制在生成代理对象的时候，应该采取哪些行为措施：

1. **proxyTargetClass（默认false）** 设置为 true 则会使用 CGLIB 对目标对象进行代理。
2. **optimize（默认false）** 用于告知代理对象是否需要才去进一步的优化措施，如代理对象生成之后，即使为其添加或者移除了相应的 Advice，代理对象也可以忽略这种变动。
3. **opaque（默认false）** 用于控制生成的代理对象是否可以强制转型为 Advised，默认是 false，表示任何生成的代理对象都可强制转型为 Advised，我们可以通过 Advised 查询代理对象的一些状态。
4. **exposeProxy（默认false）** 是否将当前代理对象绑定到 ThreadLocal。出于性能考虑，默认不绑定。如果目标对象需要访问当前代理对象，可以通过 `AopContext.currentProxy()` 取得。
5. **frozen（默认false）** 是否允许更改针对代理对象生成的各项信息配置，默认不允许，比如， ProxyFactory 设置完毕，并且 frozen 设置为 true，则不能对 Advice 进行任何变动，这样可以优化代理对象生成的性能。

要生成代理对象，只有 ProxyConfig 提供的控制信息还不够，我们还需要生成代理对象的一些具体信息，比如，要针对哪些目标类生成代理对象，要为代理对象加入何种横切逻辑等，这些信息都可以通过 `org.springframework.aop.framework.Advised` 设置或查询。默认情况下，Spring AOP 框架返回的代理对象都可以强制转型为 Advised，以查询代理对象的相关信息。

简单点说，我们可以使用 Advised 接口访问相应代理对象所持有的 Advisor，进行添加 Advisor、移除 Advisor 等相关动作。即使代理对象已经生成完毕，也可以对其进行这些操作。

AopProxy、AdvisedSupport 与 ProxyFactory 是什么关系呢？

![ProxyFactory继承层次图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ProxyFactory继承层次图.png)

ProxyFactory 集 AopProxy 和 AdvisedSupport 于一身，所以，可以通过 ProxyFactory 设置生成代理对象所需要的相关信息，也可以通过 ProxyFactory 取得最终生成的代理对象。前者是 AdvisedSupport 的职责，后者是 AopProxy 的职责。

为了重用相关逻辑，ProxyFactory 在实现的时候，将一些公用的逻辑抽取到了 `org.springframework.aop.framework.ProxyCreatorSupport` 中，它自身就继承了 AdvisedSupport，所以，生成代理对象的必要信息从其自身就可以获取到。为了简化子类生成不同类型的 AopProxy 的工作，ProxyCreatorSupport 内部持有一个 AopProxyFactory 实例，默认采用的是 DefaultAopProxyFactory，也可以通过构造方法或者是 setter 设置自定义的实现。

# ProxyCreatorSupport

ProxyFactory 只是 Spring AOP 中最基本的织入器实现，它还有另外的几个“兄弟”：

![ProxyFactory的“兄弟”](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ProxyFactory的“兄弟”.png)

# ProxyFactoryBean 介绍

在 IoC 容器中，使用 `org.springframework.aop.framework.ProxyFactoryBean` 作为织入器，它的使用与 ProxyFactory 无太大差别。ProxyFactoryBean 本质上是一个用来生产 Proxy 的 FactoryBean。如果容器中的某个对象持有某个 FactoryBean 的引用，它取得的不是 FactoryBean 本身，而是 FactoryBean 的 getObject 方法所返回的对象。所以，如果容器中某个对象依赖于 ProxyFactoryBean，那么它将会使用到 ProxyFactoryBean 的 getObject 方法所返回的代理对象。

要让 ProxyFactoryBean 的 getobject 方法返回相应目标对象类的代理对象其实很简单。因为 ProxyFactoryBean 继承了与 ProxyFactory 共有的父类 ProxyCreatorSupport，而ProxyCreatorSupport 基本上已经把要做的事情（如设置目标对象配置其他部件生成对应的 AopProxy 等）全部完成了。我们只需在 ProxyFactoryBean 的 getobject 方法中通过父类的createAopProxy 方法取得相应的 AopProxy，然后 `return AopProxy.getProxy()` 即可。

## ProxyFactoryBean 使用

ProxyFactoryBean 在继承了父类 ProxyCreatorSupport 的所有配置之外，还添加了几个自己独有的，如下所示：

- **proxyInterfaces：** 如果要采用基于接口的代理方式，可以通过改属性配置相应的接口类型。实际上，它就是调用了 `AdvisedSupport::setInterfaces` 方法，二者等价，只是使用风格不同。另外，如果目标对象实现了某个或多个接口，即使不指定要代理的接口类型，ProxyFactoryBean 也可以自动检测到目标对象所实现的接口，并对其进行基于接口的代理。
- **singleton：** 因为 ProxyFactoryBean 本质上是一个 FactoryBean，所以我们可以通过 singleton 属性，指定每次 getObject 调用是返回同一个代理对象，还是返回一个新的。通常情况下是返回同一个代理对象，即 singleton 为 true。只有在需要返回有状态的代理对象的情况下，才设置为 false，比如使用 Introduction 的场合。
- **interceptorNames：** 可以通过改属性，一次性指定多个将要织入到目标对象的 Advice、Interceptor 以及 Advisor。可以在指定的 interceptorNames 某个元素名称之后添加 “*” 通配符，可以让 ProxyFactoryBean 在容器中搜寻符合条件的所有 Advisor 并应用到目标对象。

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
