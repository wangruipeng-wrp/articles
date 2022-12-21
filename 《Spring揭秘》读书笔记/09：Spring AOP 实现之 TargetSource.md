<h1>Spring AOP 实现之 TargetSource</h1>

> 《Spring揭秘》读书笔记

- [SingletonTargetSource](#singletontargetsource)
- [PrototypeTargetSource](#prototypetargetsource)
- [HotSwappableTargetSource](#hotswappabletargetsource)
- [CommonsPoolTargetSource](#commonspooltargetsource)
- [ThreadLocalTargetSource](#threadlocaltargetsource)

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

- **getTargetClass：** 返回目标对象类型；
- **isStatic：** 用于表明是否要返回同一个目标对象实例，SingletonTargetSource 的这个方法肯定返回 true，其他的实现根据情况，通常返回 false；
- **getTarget：** 核心方法，决定返回哪个目标对象实例；
- **releaseTarget：** 释放当前调用的目标对象。但是否需要释放，完全是由实现的需要决定的，大部分时候，该方法可以只做空实现。

常见的实现类如下：

# SingletonTargetSource

`org.springframework.aop.target.SingletonTargetSource` 是使用的最多的 TargetSource 实现，通过名字都可以看出来它的实现很简单，就是内部只持有一个目标对象，当每次方法调用到达时，SingletonTargetSource 就会返回这同一个目标对象。

# PrototypeTargetSource

如果为 ProxyFactory 或者 ProxyFactoryBean 设置一个 PrototypeTargetSource 类型  TargetSource，那么每次方法调用到达调用链终点，并即将调用目标对象上的方法的时候，PrototypeTargetSource 都会返回一个新的目标对象实例供调用。但要注意目标对象的 scope 必须为 prototype。

# HotSwappableTargetSource

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

# CommonsPoolTargetSource

通过 CommonsPoolTargetSource 我们可以提供一个目标对象的对象池，然后让某个 TargetSource 实现每次都从这个目标对象池中去取得目标对象，就好像数据库连接池中的哪些 Connection 一样。CommonsPoolTargetSource 使用现有的 Jakarta Commons Pool 提供对象池支持。CommonsPoolTargetSource 使用起来跟 PrototypeTargetSource 没什么太大差别，也需要目标对象的 scope 必须为 prototype。

CommonsPoolTargetSource 还有许多控制对象池的可配置属性，比如对象池的大小、初始对象数量等。如果不能使用 Jakarta Commons Pool 对象池，那么也可以通过扩展 `org.springframework.aop.target.AbstractPoolingTargetSource`，实现相应的提供对象池化功能的 TargetSource。

# ThreadLocalTargetSource

通过 `org.springframework.aop.target.ThreadLocalTargetSource`，可以为不同的线程调用提供不同的目标对象，它可以保证各自线程上对目标对象的调用，可以被分配到当前线程对应的那个目标实现类实例上。同样，它的目标对象的 Bean 定义的 scope 也必须是 prototype。