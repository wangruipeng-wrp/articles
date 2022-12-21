<h1>IoC 和 Spring IoC</h1>

> 《Spring揭秘》读书笔记

- [IoC 概述](#ioc-概述)
  - [IoC Service Provider](#ioc-service-provider)
- [Spring IoC 概述](#spring-ioc-概述)
  - [BeanFactory](#beanfactory)
  - [ApplicationContext](#applicationcontext)

# IoC 概述

控制反转 IoC（Inversion of Control)，是一种设计思想，DI（Dependency Injection）依赖注入是实现IOC的一种方法，也有人认为DI只是IOC的另一种说法。

```java
/**
 * 新闻类
 */
public class FXNewsProvider { 

    /**
     * 监听是否发布了新的新闻
     */
    private IFXNewsListener newsListener;

    /**
     * 对新闻做持久化操作
     */
    private IFXNewsPersister newPersistener;

    public void getAndPersistNews() {
        String[] newsIds = newsListener.getAvailableNewsIds();
        if(ArrayUtils.isEmpty(newsIds)) {
            return;
        }
    
        for(String newsId : newsIds) { 
            FXNewsBean newsBean = newsListener.getNewsByPK(newsId);
            newPersistener.persistNews(newsBean);
            newsListener.postProcessIfNecessary(newsId);
        }
    } 
}
```

上面的新闻类中需要 `newsListener` 监听新的新闻消息，`newPersistener` 对新闻做持久化操作，于是新闻类的内部需要依赖这两个对象才能完成新闻的获取和持久化操作。那么新闻类的构造器就应该是这样：

```java
public FXNewsProvider() {
    this.newsListener = new NewsListener();
    this.newPersistener = new NewsPersister();
}
```

但是回头看看我们是不是需要这么做呢？我们最终所要做的，其实就是直接调用 `persistNews()` 和 `postProcessIfNecessary()`，但是为了这两个方法我们需要构造整个对象出来，相当于加载了这个对象里面的全部方法和属性，这很浪费。所以我们还能这样：

```java
public FXNewsProvider(NewsListener newsListener, NewsPersister newsPersister) { 
    this.newsListener = newsListener;
    this.newPersistener = newsPersister;
}
```

让其他人去构造这两个对象，新闻类直接用就好了。所以，简单点来说，**IoC的理念就是：让别人为你服务！**

通常情况下，被注入对象会直接依赖于被依赖对象。但是，在IoC的场景中，二者之间通过IoC Service Provider来打交道，所有的被注入对象和依赖对象现在由IoC Service Provider统一管理。

被注入对象需要什么，直接跟IOC Service Provider招呼一声，后者就会把相应的被依赖对象注入到被注入对象中，从而达到IoC Service Provider为被注入对象服务的目的。IoC Service Provider在这里就是通常的IoC容器所充当的角色。

从被注入对象的角度看，与之前直接寻求依赖对象相比，依赖对象的取得方式发生了**反转**，控制也从被注入对象转到了IoC Service Provider那里。

> 如果要用一句话来概括 IoC 可以给我们带来什么，那么我希望是，IoC 是一种可以帮助我们解耦各业务对象间依赖关系的对象绑定方式！

## IoC Service Provider

虽然业务对象可以通过 IoC 方式声明相应的依赖（构造器、setter、属性等注入方式），但是最终仍然需要通过某种角色或者服务将这些对象绑定到一起，IoC Service Provider 就对应着 IoC 场景中的这一角色。

IoC Service Provider 的职责相对来说比较简单，主要有两个：**业务对象的构建管理**和**业务对象间的依赖绑定**。

业务对象的构建管理由容器内部自动搞定，但是业务对象间的依赖绑定关系就需要程序员来描述了。可以使用直接编码、配置文件、注解等方式描述业务对象之间的依赖绑定关系。

# Spring IoC 概述

Spring 的 IoC 容器就是一个 IoC Service Provider，但 Spring 的 IoC 容器还提供了一些额外的服务。例如：对象生命周期管理、对象作用域范围管理、AOP支持等。

Spring 提供了两种容器类型：**BeanFactory** 和 **ApplicationContext**。

## BeanFactory

基础类型 IoC 容器，提供完整的 IoC 服务支持，默认采用延迟初始化策略。**所以，相对来说，容器启动初期速度较快，所需要的资源有限。** 对于资源有限，并且功能要求不是很严格的场景，BeanFactory是比较合适的 IoC 容器选择。

## ApplicationContext

ApplicationContext 在 BeanFactory 的基础上构建，是相对比较高级的容器实现，除了拥有 BeanFactory 的所有支持，ApplicationContext 还提供了其他高级特性，比如事件发布、国际化信息支持等。

ApplicationContext 所管理的对象，在该类型容器启动之后，默认全部初始化并绑定完成。所以，相对于 BeanFactory 来说，ApplicationContext 要求更多系统资源，启动时间也会更长一些。在哪些系统资源充足，并且要求更多功能的场景中，ApplicationContext 类型的容器是比较合适的选择。

![ApplicationContext和BeanFactory继承关系图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ApplicationContext和BeanFactory继承关系图.png)

