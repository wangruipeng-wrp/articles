---
title: Spring的IoC容器之 ApplicationContext
abbrlink: 11908
date: 2022-11-16 23:15:56
description: 《Spring揭秘》读书笔记
---

**常用实现：**

- `org.springframework.context.support.FileSystemXmlApplicationContext` 在默情况下，从文件系统加载 Bean 定义以及相关资源的 ApplicationContext 实现。
- `org.springframework.context.support.classPathXmlApplicationContext` 在默认情况下，从 Classpath 加载bean定义以及相关资源的 ApplicationContext 实现。
- `org.springframework.web.context.support.XmlWebApplicationContext` Spring 提供的用于 Web 应用程序的 ApplicationContext 实现。

作为 Spring 提供的较之 BeanFactory 更为先进的 IoC 容器实现，ApplicationContext 除了拥有 BeanFactory 支持的所有功能之外，还有一些特有的特性，即统一资源加载策略、国际化信息支持、容器内事件发布、多配置模块加载简化等。

# 统一资源加载策略

Spring 框架内部使用 `org.springframework.core.io.Resource` 接口作为所有资源的抽象和访问接口，就是说它用来**描述资源**，在构造 BeanFactory 的时候已经接触过了，如下：

```java
BeanFactory beanFactory = new XmlBeanFactory(new ClassPathResource("..."));
```

其中 ClassPathResource 就是 Resource 的一个特定类型的实现，代表的是位于 Classpath 中的资源。Resource 接口还有一些其他实现类：

- `ByteArrayResource` 将字节数组提供的数据作为一种资源进行封装，如果通过 InputStream 形式访问该类型的资源，该实现会根据字节数组的数据，构造相应的 ByteArrayInputstream 并返回。
- `ClassPathResource` 该实现从Java应用程序的 ClassPath 中加载具体资源并进行封装，可以使用指定的类加载器或者给定的类进行资源加载。
- `FileSystemResource` 对java.io.File类型的封装，所以，我们可以以文件或者URL的形式对该类型资源进行访问，只要能跟File打的交道，基本上跟FileSystemResource也可以。
- `UrlResource` 通过 java.net.URL 进行的具体资源查找定位的实现类，内部委派 URL 进行具体的资源操作。

## ResourceLoader

ResourceLoader 职责：查找和定位资源。`org.springframework.core.io.ResourceLoader` 接口是资源查找定位策略的统一抽象，具体的资源查找定位策略则由相应的 ResourceLoader 实现类给出。

```java
public interface ResourceLoader { 
    String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;
    Resource getResource(String location);
    ClassLoader getClassLoader();
}
```

**DefaultResourceLoader(class implements ResourceLoader)**

ResourceLoader 有一个默认的实现类，即 `org.springframework.core.io.DefaultResourceLoader`，该类默认的资源查找处理逻辑如下：

1. 首先，检查资源路径是否以 `classpath:` 前缀打头，是的话构造 ClassPathResource 类型资源并返回；
2. 否则，通过 URL，根据资源路径来定位资源，如果没抛出 `MalformedURLException` 则构造 UrlResource 类型资源并返回；
3. 否则，委派给 `getResourceByPath(String)` 方法，该方法的默认实现是，构造 ClassPathResource 类型的资源并返回；
4. 最终，如果还没找到，`getResourceByPath(String)` 方法会构造一个不存在的资源并返回。

**FileSystemResourceLoader(class extends DefaultResourceLoader)**

继承自 DefaultResourceLoader，并重写了 `getResourceByPath(String)` 方法。使之从文件系统加载资源并以 FileSystemResource 类型返回。

**ResourcePatternResolver(interface extends ResourceLoader)**

是 ResourceLoader 的扩展，可以根据指定的资源路径匹配模式每次返回多个 Resource 实例。而 ResourceLoader 每次只能返回单一实例。接口定义如下：

```java
public interface ResourcePatternResolver extends ResourceLoader {
    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";
    Resource[] getResources(String locationPattern) throws IOException;
}
```

**PathMatchingResourcePatternResolver(class implements ResourcePatternResolver)**

构造该实例时，可以指定一个 ResourceLoader，默认是 DefaultResourceLoader。如果不指定任何 ResourceLoader 的话，`PathMatchingResourcePatternResolver` 在加载资源的行为上会与 DefaultResourceLoader 基本相同，只存在返回的 Resource 数量上的差异。

以上 ResourceLoader 之间的继承关系：

![ResourceLoader间继承关系](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ResourceLoader间继承关系.png)

## ApplicationContext与ResourceLoader

ApplicationContext 继承了 ResourcePatternResolver，当然就间接实现了 ResourceLoader 接口，所以，任何的 ApplicationContext 实现都可以看作是一个 ResourceLoader 甚至是 ResourcePatternResolver。而这就是 ApplicationContext 支持 Spring 内统一资源加载策略的真相。

通常，所有的 ApplicationContext 实现类会直接或者间接地继承 `org.springframework.context.support.AbstractApplicationContext`，从这个类上，我们就可以看到 ApplicationContext 与 ResourceLoader 之间的所有关系。

AbstractApplicationContext 继承了 DefaultResourceLoader，那么，它的 getResource(String) 当然就直接用 DefaultResourceLoader 的了。剩下还需要它做的就是 ResourcePatternResolver 的 `Resource[] getResources(String)` 了。

AbstractApplicationContext 类的内部声明有一个 resourcePatternResolver 属性，对应的实例类型为 PathMatchingResourcePatternResolver。PathMatchingResourcePatternResolver 在构造时会接收一个 ResourceLoader，而 AbstractApplicationContext 又继承自 DefaultResourceLoader，当然直接把自身注入进 PathMatchingResourcePatternResolver 了，而不是重新构造 DefaultResourceLoader。

> **所以：**`AbstractApplicationContext` 即可作为 `ResourceLoader` 又可以作为 `ResourcePatternResolver`

如果某个类需要依赖到 ResourceLoader 来查找定位资源，当然就可以直接把 ApplicationContext 容器实例注入到该类中。Spring 为这提供了 ResourceLoaderAware 和 ApplicationContextAware 接口。在 Bean 的生命周期中会可以为对象实例注入 ApplicationContext。

当然要求业务对象要实现 Spring 的接口就过于依赖 Spring 框架了，但我觉得这没什么不好的。



# 容器内事件发布

Java SE 提供了实现自定义事件发布功能的基础类，即 `java.util.EventObject` 类和 `java.util.EventListener` 接口。所有的自定义事件类型可以通过扩展 EventObject 来实现，而事件监听器则可以实现 EventListener 接口。使用起来可以像这样：

## ApplicationContext 事件发布

Spring 的 ApplicationContext 容器内部允许以 `org.springframework.context.ApplicationEvent` 的形式发布事件，容器内注册的 `org.springframework.context.ApplicationListener` 类型的 Bean 定义会被 ApplicationContext 容器自动识别，它们负责监听容器内发布的所有 ApplicationEvent 类型的事件。也就是说，一旦容器内发布 ApplicationEvent 及其子类型的事件，注册到容器的 ApplicationListener 就会对这些事件进行处理。

**ApplicationEvent(abstract class)**

继承自 `java.util.EventObject` 是一个抽象类，需要根据情况提供相应子类以区分不同情况。Spring 提供了三个实现：

1. ContextClosedEvent：ApplicationContext 容器在即将关闭的时候发布的事件类型。
2. ContextRefreshedEvent：ApplicationContext 容器在初始化或者刷新的时候发布的事件类型。
3. RequestHandlerEvent：Web 请求处理后发布的事件，其有一子类 ServletRequestHandlerEvent 提供特定于 Java EE 的 Servlet 相关事件。

**ApplicationListener(interface)**

ApplicationContext 容器内使用的自定义事件监听器接口定义，继承自 `java.util.EventListener`。ApplicationContext 容器在启动时，会自动识别并加载 EventListener 类型 Bean定义，一旦容器内有事件发布，将通知这些注册到容器的 EventListener。

**ApplicationContext**

ApplicationContext 继承了 ApplicationEventPublisher 接口，该接口提供了 `void publishEvent(Object event)` 方法定义，自然也能充当事件发布者的角色。

ApplicationContext 容器把事件的发布和事件监听器的注册这些事情给转包给 `org.springframework.context.event.ApplicationEventMulticaster` 接口来实现，该接口定义了具体事件监听器的注册管理以及事件发布的方法，它有一个抽象实现类 `org.springframework.context.event.AbstractApplicationEventMulticaster`，这个抽象类实现了事件监听器的管理功能，出于灵活性和扩展性考虑，事件的发布功能则委托给了其子类。Spring 为这个抽象类提供了一个子类实现 `org.springframework.context.event.SimpleApplicationEventMulticaster`，添加了事件发布功能。不过，其默认使用了 SyncTaskExecutor 进行事件的发布，这可能存在一些性能问题，可以为其提供其他类型的 TaskExecutor 实现类。

因为 ApplicationContext 容器的事件发布功能全部委托给了 ApplicationEventMulticaster，所以在容器启动的时候就会检查容器内是否存在名称为 applicationEventMulticaster 的 ApplicationEventMulticaster 对象实例。有的话就使用提供的实现，没有则默认初始化一个 SimpleApplicationEventMulticaster 作为默认实现。

![ApplicationContext事件发布关系图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ApplicationContext事件发布关系图.png)

如果在业务对象中需要使用到事件发布，可以实现 `ApplicationEventPublisherAware` 接口或者是实现 `ApplicationContextAware` 接口。

---

> ApplicationContext 还有国际化信息支持、多配置模块加载简化等其他功能，但是平时用的比较少，这里也就不多说了。实际上本文提到的统一资源加载策略、容器内事件发布等功能在平时用的也不多，但是这一块内容 Spring 在接口、类的继承和实现关系上处理的比较复杂，值得学习与研究。