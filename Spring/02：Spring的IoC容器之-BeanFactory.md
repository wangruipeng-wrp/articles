---
title: Spring的IoC容器之 BeanFactory
abbrlink: 55764
date: 2022-11-16 23:15:39
description: 《Spring揭秘》读书笔记
---

# BeanFactory

BeanFactory，顾名思义，就是生产 Bean 的工厂。作为 Spring 提供的基本的 IoC 容器，BeanFactory 可以完成作为 IoC Service Provider 的所有职责，包括业务对象的注册和对象间依赖关系的绑定。

```java
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";
    Object getBean(String name) throws BeansException;
    Object getBean(String name, Class requiredType) throws BeansException;
    Object getBean(String name, Object[] args) throws BeansException;
    boolean containsBean(String name);
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, Class targetType) throws NoSuchBeanDefinitionException;
    Class getType(String name) throws NoSuchBeanDefinitionException;
    // 获取别名
    String[] getAliases(String name);
}
```

之前我们的系统业务对象需要自己去“拉”（PULL）所依赖的业务对象，有了 BeanFactory 之类的 IoC 容器之后，需要依赖什么让 BeanFactory 为我们“推”（PUSH）过来就行了。

- 自己去“拉”就相当于是自己创建所依赖的对象。
- 让 BeanFactory 为我们“推”过来就必须把依赖对象和被依赖对象都交给 BeanFactory 去管理。

像这样：

```java
BeanFactory container = new XmlBeanFactory(new ClassPathResource("配置文件路径"));
FXNewsProvider newsProvider = (FXNewsProvider) container.getBean("BeanName");
newsProvider.getAndPersistNews();
```

虽然把对象直接交给 BeanFactory 管理，但是对象之间的依赖绑定还是需要程序员自己描述清楚才可以。可以直接编码来描述对象之间的依赖关系，也可以使用配置文件来描述比如 .xml 文件，或者是注解比如 `@Autowired` 和 `@Component` ，但不管是什么方式，最终都会转换成代码的方式运行。

像这样：

```java
public static void main(String[] args) {
    DefaultListableBeanFactory beanRegistry = new DefaultListableBeanFactory();
    BeanFactory container = (BeanFactory) bindViaCode(beanRegistry);
    FXNewsProvider newsProvider = (FXNewsProvider) container.getBean("newsProvider");
    newsProvider.getAndPersistNews();
}

public static BeanFactory bindViaCode(BeanDefinitionRegistry registry) {
    AbstractBeanDefinition newsProvider  = new RootBeanDefinition(FXNewsProvider.class, true);
    AbstractBeanDefinition newsListener  = new RootBeanDefinition(NewsListener.class, true);
    AbstractBeanDefinition newsPersister = new RootBeanDefinition(NewsPersister.class,true);
 
    // 将 Bean 定义注册到容器中
    registry.registerBeanDefinition("newsProvider", newsProvider);
    registry.registerBeanDefinition("listener", newsListener);
    registry.registerBeanDefinition("persister", newsPersister);

    // 指定依赖关系
    // 1. 可以通过构造方法注入方式
    ConstructorArgumentValues argValues = new ConstructorArgumentValues();
    argValues.addIndexedArgumentValue(0, newsListener);
    argValues.addIndexedArgumentValue(1, newsPersister);
    newsProvider.setConstructorArgumentValues(argValues);

    // 2. 或者通过 setter 方法注入方式
    MutablePropertyValues propertyValues = new MutablePropertyValues();
    propertyValues.addPropertyValue(new PropertyValue("newsListener",newsListener));
    propertyValues.addPropertyValue(new PropertyValue("newPersistener",newsPersister));
    newsProvider.setPropertyValues(propertyValues);

    // 绑定完成
    return (BeanFactory) registry;
} 
```

BeanFactory 只是一个接口，我们最终需要一个该接口的实现来进行实际的 Bean 的管理，`DefaultListableBeanFactory` 就是这么一个比较通用的 BeanFactory 实现类。

`DefaultListableBeanFactory` 除了间接实现了 BeanFactory 接口，还实现了 `BeanDefinitionRegistry` 接口，该接口才是 BeanFactory 的实现中担当 Bean 注册管理的角色。BeanFactory 接口只定义如何访问容器内管理 Bean 的方法，各个 BeanFactory 的具体实现类负责具体 Bean 的注册以及管理工作。

`BeanDefinitionRegistry` 接口定义抽象了 Bean 的注册逻辑。通常情况下，具体的 BeanFactory 实现类会实现这个接口来管理 Bean 的注册。它们之间的关系像这样：

![BeanDefinitionRegistry关系图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/BeanDefinitionRegistry关系图.jpg)

每一个受管的对象，在容器中都会有一个 BeanDefinition 的实例与之相对应，该 BeanDefinition 的实例负责保存对象的所有必要信息，包括其对应的对象的class类型、是否是抽象类、构造方法参数以及其他属性等。当客户端向 BeanFactory 请求相应对象的时候，BeanFactory 会通过这些信息为客户端返回一个完备可用的对象实例。

# Bean 的 scope

Spring 容器最初提供了两种 Bean 的 scope 类型：singleton（默认） 和 prototype，但发布 2.0 之后，又引入了另外三种类型，即 request、session 和 global session 类型。不过这三种类型有所限制，只能在 Web 应用中使用。也就是说，只有在支持 Web 应用的 ApplicationContext 中使用这三个 scope 才是合理的。

## singleton

标记为拥有 singleton scope 的对象定义，在 Spring 的 IoC 容器中只存在一个实例，所有对该对象的引用将共享这个实例。该实例从容器启动，并因为第一次被请求而初始化之后，将一直存活到容器退出，也就是说，它与 IoC 容器“几乎”拥有相同的“寿命”。

注意：标记为 singleton 的 bean 是由容器来保证这种类型的 bean 在同一个容器中只存在一个共享实例。如果你要手动创建它那容器也管不着是吧，应该要与单例模式区分开来，单例模式是保证在同一个 ClassLoader 中只存在一个这种类型的实例。

## prototype

针对声明为拥有 prototype scope 的 Bean 定义，容器在接到该类型对象的请求的时候，会每次都重新生成一个新的对象实例给请求方。虽然这种类型的对象的实例化以及属性设置等工作都是由容器负责的，但是只要准备完毕，并且对象实例返回给请求方之后，容器就不再拥有当前返回对象的引用，请求方需要自己负责当前返回对象的后继生命周期的管理工作，包括该对象的销毁。也就是说，容器每次返回给请求方一个新的对象实例之后，就任由这个对象实例“自生自灭”了。

## 自定义 scope

`org.springframework.beans.factory.config.Scope` 接口：

```java
public interface Scope { 
    Object get(String name, ObjectFactory objectFactory);
    Object remove(String name);
    void registerDestructionCallback(String name, Runnable callback);
    String getConversationId();
}
```

要实现自己的 scope 类型，首先需要给出一个 Scope 接口的实现类，接口定义中的4个方法并非都是必须的，但 `get()` 和 `remove()` 方法必须实现。一个基于线程的 ThreadScope 的实现：

```java
public class ThreadScope implements Scope {
    private final ThreadLocal<Map<String, Object>> threadScope = ThreadLocal.withInitial(HashMap::new);

    /**
     * 容器每次获取对象会来调用这个方法
     */
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> scope = threadScope.get();
        Object obj = scope.get(name);
        if (obj == null) {
            obj = objectFactory.getObject();
            scope.put(name, obj);
        }
        return obj;
    }

    public Object remove(String name) {
        Map<String, Object> scope = threadScope.get();
        return scope.remove(name);
    }

    // ...
}
```

有了 Scope 的实现类之后，我们需要把这个 Scope 注册到容器中，才能提供相应的 Bean 定义使用。通常情况下，我们可以使用 `ConfigurableBeanFactory` 的 `void registerScope(String scopeName, Scope scope);` 方法注册自定义 Scope。

`ConfigurableBeanFactory` 接口间接继承了 BeanFactory 接口。

Spring 还提供了一个 `BeanFactoryPostProcessor` 用于注册自定义 Scope，即 `org.springframework.beans.factory.config.CustomScopeConfigurer`。

对于 ApplicationContext 来说，它可以自动识别并加载 `BeanFactoryPostProcessor`，所以可以直接配置 `CustomScopeConfigurer`。像这样：

```java
@Bean
CustomScopeConfigurer csc() {
    CustomScopeConfigurer c = new CustomScopeConfigurer();
    c.addScope("thread", new ThreadScope());
    return c;
}
```

# BeanFactory 运行阶段

Spring 的 IoC 容器实现以上功能的过程，基本上可以按照类似的流程划分为两个阶段，即**容器启动阶段**和**Bean实例化阶段**

## 容器启动阶段

> .xml, @Component --> BeanDefinition

容器启动伊始，首先会通过某种途径加载 Configuration MetaData。除去代码方式比较直接，在大部分情况下，容器需要依赖某些工具类（BeanDefinitionReader）对加载的 Configuration MetaData 进行解析和分析，并将分析后的信息编组为相应的 BeanDefinition，最后把这些保存了 Bean 定义必要信息的 BeanDefinition，注册到相应的 BeanDefinitionRegistry，这样容器的启动工作就完成了。

Spring 提供了一种叫做 BeanFactoryPostProcessor 的容器扩展机制。该机制允许我们在容器实例化相应对象之前，对注入到容器的 BeanDefinition 所保存的信息做相应的修改。这可以让我们对最终的 BeanDefinition 做一些额外的操作，比如修改其中 Bean 定义的某些属性，为 Bean 定义增加其他信息等。

如果要自定义实现 BeanFactoryPostProcessor，需要实现 `org.springframework. beans.factory.config.BeanFactoryPostProcessor` 接口。又因为一个容器可能拥有多个 BeanFactoryPostProcessor，可能还需要实现 `org.springframework.core.Ordered` 接口来保证各个 BeanFactoryPostProcessor 可以按照顺序执行。

**常见的 BeanFactoryPostProcessor ：**

- `org.springframework.beans.factory.config.PropertyPlaceholderConfigurer` 处理配置文件中的占位符
- `org.springframework.beans.factory.config.PropertyOverrideConfigurer` 替换 Bean 定义的属性信息
- `org.springframework.beans.factory.config.CustomEditorConfigurer` 用于注册一些自定义的字符串转换，例如：`StringArrayPropertyEditor` 将以逗号分隔的字符串转换成字符数组。如果需要自定义字符串到Java类型的转换，则需要实现 `PropertyEditor` 接口，再通过`CustomEditorConfigurer` 注册自定义的 `PropertyEditor`。但通常可以直接继承 `java.beans.PropertyEditorSupport` 再重写相关方法，会更方便一些。

## Bean实例化阶段

当某个请求方通过容器的 getBean 方法明确地请求某个对象，或者因依赖关系容器需要隐式地调用 getBean 方法时，就会触发第二阶段的活动。

该阶段，容器会首先检查所请求的对象之前是否已经初始化。如果没有，则会根据注册的 BeanDefinition 所提供的信息实例化被请求对象，并为其注入依赖。如果该对象实现了某些回调按口，也会根据回调接口的要求来装配它。当该对象装配完毕之后，容器会立即将其返回请求方使用。

Bean 的实例化过程像这样：

![Bean实例化过程](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/Bean实例化过程.jpg)

Spring 容器将对其所管理的对象全部给予统一的生命周期管理，这些被管理的对象完全摆脱了原来那种 “new 完后被使用，脱离作用于后即被回收” 的命运。

### Bean的实例化与BeanWrapper

容器在内部实现的时候，采用策略模式来决定采用何种方式初始化 bean 实例。通常，可以通过反射或者 CGLIB 动态字节码生成来初始化相应的 bean 实例或者动态生成其子类。

- 反射：`SimpleInstantiationStrategy`
- CGLIB：`CglibSubclassingInstantiationStrategy`，容器默认的初始化方式。

容器只要根据相应 Bean 定义的 BeanDefinition 取得实例化信息，结合 CglibSubclassingInstantiationStrategy 以及不同的 Bean 定义类型，就可以返回实例化完成的对象实例。但不是直接返回构造完成的对象实例，而是以 BeanWrapper 对构造完成的对象实例进行包裹，返回相应的 BeanWrapper 实例。

**至此，第一步结束。**

BeanWrapper 的作用是对某个 Bean 进行“包裹”，然后对这个“包裹”的 Bean 进行操作，比如设置或者获取 Bean 的属性值。第二步的设置属性值，就在这里完成。

前文提到的 `CustomEditorConfigurer`，当把各种各样的 `PropertyEditor` 注入到容器中后，后面就是由 BeanWrapper 来使用这些 `PropertyEditor`。

使用 BeanWrapper 设置属性的方式，免去了反射设置属性的繁琐：

```java
// 这里除了使用反射也可以使用 CGLIB
Object provider  = Class.forName("package.name.FXNewsProvider").newInstance();
Object listener  = Class.forName("package.name.NewsListener").newInstance();
Object persister = Class.forName("package.name.NewsPersister").newInstance();

BeanWrapper newsProvider = new BeanWrapperImpl(provider);
newsProvider.setPropertyValue("newsListener", listener);
newsProvider.setPropertyValue("newPersistener", persister);

assertTrue(newsProvider.getWrappedInstance() instanceof FXNewsProvider); 
assertSame(provider, newsProvider.getWrappedInstance()); 
assertSame(listener, newsProvider.getPropertyValue("newsListener")); 
assertSame(persister, newsProvider.getPropertyValue("newPersistener"));
```

### 各色的 Aware 接口

当对象实例化完成并且相关属性以及依赖设置完成之后，Spring 容器会检查当前对象实例是否实现了一系列的以 Aware 命名结尾的接口定义。如果是，则将这些 Aware 接口定义中规定的依赖注入给当前对象实例。有这些 Aware 接口：

- `org.springframework.beans.factory.BeanNameAware` 将该对象实例的 Bean 定义对应的 BeanName 设置到当前对象实例。
- `org.springframework.beans.factory.BeanClassLoaderAware` 将对应加载当前 Bean 的 ClassLoader 注入当前对象实例。
- `org.springframework.beans.factory.BeanFactoryAware` 将 BeanFactory 容器自身设置到当前对象实例。

### BeanPostProcessor

> BeanPostProcessor 是容器提供的对象实例化阶段的强有力的扩展点。

```java
package org.springframework.beans.factory.config;

public interface BeanPostProcessor {
    /**
     * 前置处理要执行的方法
     */
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    /**
     * 后置处理要执行的方法
     */
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

比较常见的 BeanPostProcessor 的场景：

- 为当前对象提供代理
- 处理标记接口实现类：ApplicationContext 对应的哪些 Aware 接口实际上就是通过 BeanPostProcessor 的方式进行处理的。

### InitializingBean和@PostConstruct

```java
public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
```

该接口的作用在于，在对象实例化过程调用过 “BeanPostProcessor 的前置处理” 之后，会接着检测当前对象是否实现了 InitializingBean 接口，如果是，则会调用其 afterPropertiesSet() 方法进一步调整对象实例的状态。比如，在有些情况下，某个业务对象实例化后，还不能处于可以使用状态。这个时候就可以让该业务对象实现该接口，并在方法 afterPropertiesSet() 中完成对该业务对象的后续处理。

如果让业务对象实现这个接口，则显得 Spring 容器比较具有侵入性。所以，Spring 还提供了另一种方式来指定自定义的对象初始化操作，`@PostConstruct` 注解。通过 `@PostConstruct` 注解，业务对象的自定义初始化操作可以以任何方式命名，而不再受制于 afterPropertiesSet()。

### DisposableBean与@PreDestroy

```java
public interface DisposableBean {
	void destroy() throws Exception;
}
```

与 InitializingBean 和 @PostConstruct 相对应，DisposableBean 与 @PreDestroy 为对象提供了执行自定义销毁逻辑的机会。

最常见到该功能的使用场景就是在 Spring 容器中注册数据库连接池，在系统推出后，连接池应该关闭，以释放相应资源。