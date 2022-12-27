---
title: Spring Security
abbrlink: 28582
date: 2022-09-06 11:38:08
tags:
---

关于 Spring Security 的文章网上有很多，B站也有很多教程，我都看过一些，但感觉对这部分的知识还不是很牢固。网上的大部分文章主要都是从后端的用户认证开始讲起的，一上来就是这些比较偏向实用部分的内容我觉得不是很好理解，还有很多巨长的类名方法名等等，这并不是在说网上的这些文章写的不好，毕竟本文也是一篇前人栽树，后人乘凉的文章。但是写这篇文章的初衷是想要从一个简单的小例子入手，解释清楚那些巨长的类名方法名都是干嘛的，应该怎么使用。

Spring Security 的一些介绍就不多说了，直接一个小 demo 走起，这个 demo 力争简洁，代码如下：

```java
public class Demo {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.println("请输入账号：");
        String username = sc.next();
        System.out.println("请输入密码：");
        String password = sc.next();

        try {
            // 开始认证    
            final AuthenticationManager am = new SampleAuthenticationManager();

            // 1、封装一个 UsernamePasswordAuthenticationToken 对象
            final Authentication unAuth = new UsernamePasswordAuthenticationToken(username, password);

            // 2、经过 AuthenticationManager 的认证，如果认证失败会抛出一个 AuthenticationException 错误
            final Authentication auth = am.authenticate(unAuth);

            // 3、将这个认证过的 Authentication 填入 SecurityContext 里面
            SecurityContextHolder.getContext().setAuthentication(auth);
        } catch (AuthenticationException e) {
            System.out.println("认证失败：" + e.getMessage());
        }

        // 从 SecurityContext 中取出 Authentication
        final Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        System.out.println("认证成功！");
        System.out.println("输入的账号：" + auth.getName());
        System.out.println("输入的密码：" + auth.getCredentials());
    }
}

// 实现一个简单 AuthenticationManager 用于认证
class SampleAuthenticationManager implements AuthenticationManager {

    // 关键认证部分
    @Override
    public Authentication authenticate(Authentication auth) throws AuthenticationException {

        String username = auth.getName();
        String password = (String) auth.getCredentials(); // getCredentials 返回的是密码

        if (username == null || "".equals(username))
            throw new BadCredentialsException("账号不能为空");
        if (password == null || "".equals(password))
            throw new BadCredentialsException("密码不能为空");

        // 认证成功，创建认证成功对象
        return new UsernamePasswordAuthenticationToken(auth.getName(), auth.getCredentials());
    }
}
```

Spring Security 主要的两个功能是认证和鉴权。这两个功能组合起来就能完成用户登录的需求，也就是一个系统中最基础最重要的模块。根据不同系统的需求，用户认证的工作可能会很繁琐。具体的认证步骤由程序员自行编写，Spring Security 的策略是把每个认证步骤串联起来成为一整个认证流程。上面的这个 demo 仅仅是一个简单的账号密码判空操作而已。主要是说明了怎么编写认证步骤。

# 重要组件

- Authentication：认证对象接口，定义了认证对象的数据形式。
- AuthenticationManager：认证工作的上层接口，用于校验 `Authentication`，返回一个认证完成后的 `Authentication` 对象。
- SecurityContext：上下文对象，`Authentication` 对象会放在里面。
- SecurityContextHolder：用于拿到上下文对象的静态工具类。

## Authentication

Authentication 只是定义了一种在 Spring Security 进行认证过的数据的数据形式应该是怎么样的，要有权限，要有密码，要有身份信息，要有额外信息。

```java
public interface Authentication extends Principal, Serializable {
    // 获取用户权限列表
    Collection<? extends GrantedAuthority> getAuthorities();
    // 获取证明用户认证的信息，通常情况下获取到的是密码等信息。
    Object getCredentials();
    // 获取用户的额外信息，（这部分信息可以是我们的用户表中的信息）。
    Object getDetails();
    // 获取用户身份信息，在未认证的情况下获取到的是用户名，在已认证的情况下获取到的是 UserDetails。
    Object getPrincipal();
    // 获取当前 Authentication 是否已认证。
    boolean isAuthenticated();
    // 设置当前 Authentication 是否已认证（true or false）。
    void setAuthenticated(boolean var1) throws IllegalArgumentException;
}
```

## AuthenticationManager

`AuthenticationManager` 定义了一个认证方法，它将一个未认证的 `Authentication` 传入，返回一个已认证的 `Authentication`，默认使用的实现类为：`ProviderManager`。

```java
public interface AuthenticationManager {
    // 认证方法
    Authentication authenticate(Authentication authentication) 
        throws AuthenticationException;
}
```

## SecurityContext

上下文对象，认证后的数据就放在这里面，这个接口里面只有两个方法，其主要作用就是 get or set `Authentication`。接口定义如下：

```java
public interface SecurityContext extends Serializable {
    Authentication getAuthentication(); // 获取Authentication对象
    void setAuthentication(Authentication authentication); // 放入Authentication对象
}
```

## SecurityContextHolder

可以说是 `SecurityContext` 的工具类，用于 get or set or clear `SecurityContext`，默认会把数据都存储到当前线程中。

```java
public class SecurityContextHolder {
    public static void clearContext() {
        strategy.clearContext(); 
    }
    public static SecurityContext getContext() {
        return strategy.getContext();
    }
    public static void setContext(SecurityContext context) {
        strategy.setContext(context);
    }
}
```

---

由这四个组件组成的认证流程就是：

1. 将登录请求的身份信息封装成未认证的 `Authentication`
2. 将未认证的 `Authentication` 交由 `AuthenticationManager` 实行认证
3. `AuthenticationManager` 认证完成返回认证后的 `Authentication`
4. 将认证过的 `Authentication` 放入 `SecurityContext`

上面的这四个组件就是 Spring Security 当中最重要的几个组件，Spring Security 其他的内容也是围绕这几个组件展开的。

# ProviderManager

ProviderManager 是 AuthenticationManager 的默认实现类，由它衍生出来了 AuthenticationProvider 接口。

原来的一个 AuthenticationManager 只做一次认证工作，但是 ProviderManager 把多个认证工作放在一个集合中，遍历取出每个认证对象一次次做认证工作，只要有一次通过了，就认为这次认证是成功的。在 ProviderManager 中把认证工作封装成了 AuthenticationProvider。

{% note default %}
**AuthenticationProvider：**
{% endnote %}

认证提供者，服务于 ProviderManager 类，由实现了这个接口的对象组成一个集合，在 ProviderManager 中遍历取出认证 Authentication。

```java
public interface AuthenticationProvider {

    // 认证
    Authentication authenticate(Authentication authentication)
            throws AuthenticationException;

    // 是否支持当前的 Authentication 对象
    boolean supports(Class<?> authentication);
}
```

ProviderManager 这个类的代码比较复杂，摘一些跟它认证流程有关的代码出来看看，以下代码在源码的基础上有做改动，主要是为了方便看一些

```java
/* 构造方法之一，传入一个个的 AuthenticationProvider 对象
 * 待会就拿这些对象来认证。
 * parent 对象传入 null，parent 相当于是一个兜底的 AuthenticationManager，
 * 如果所有的 AuthenticationProvider 认证都没通过，则使用 parent 做一次认证。
 */
public ProviderManager(AuthenticationProvider... providers) {
    this(Arrays.asList(providers), (AuthenticationManager)null);
}

public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    Class<? extends Authentication> toTest = authentication.getClass();
    Authentication result = null;
    Authentication parentResult = null;

    // 依次取出 provider 认证 authentication
    for(AuthenticationProvider provider : this.getProviders()) {
        // provider 不能支持 authentication，直接跳过
        if (!provider.supports(toTest))  continue;

        try {
            // 调用 provider 进行认证
            result = provider.authenticate(authentication);
            // 认证通过
            if (result != null) {
                this.copyDetails(authentication, result);
                break;
            }
        } catch (InternalAuthenticationServiceException | AccountStatusException | AuthenticationException e) {
            // 某次异常处理
        }
    }

    // providers 认证失败，尝试使用 parent 认证
    if (result == null && this.parent != null) {
        try {
            result = parentResult = this.parent.authenticate(authentication);
        } catch (ProviderNotFoundException | AuthenticationException e) {
            // 异常处理
        }
    }

    if (result != null) {
        // 认证成功，擦除密码信息
        if (this.eraseCredentialsAfterAuthentication && result instanceof CredentialsContainer) {
            ((CredentialsContainer)result).eraseCredentials();
        }
        // 发布认证成功事件
        if (parentResult == null) {
            this.eventPublisher.publishAuthenticationSuccess(result);
        }
        return result;
    } else { 
        // 执行至此，说明 providers 和 parent 都没认证成功，包装异常信息抛出
    }
}
```

上面这段代码如果觉得比较复杂也可以不看，只需知道 ProviderManager 类的认证流程即可。

# DaoAuthenticationProvider

DaoAuthenticationProvider 就是 AuthenticationProvider 的最常实现类，顾名思义，Dao 正是数据访问层的缩写，也暗示了这个身份认证器的实现思路。

按照我们最直观的思路，怎么去认证一个用户呢？

用户前台提交了用户名和密码，而数据库中保存了用户名和密码，认证便是负责比对同一个用户名，提交的密码和保存的密码是否相同便是了。

在 Spring Security 中。提交的用户名和密码，被封装成了 UsernamePasswordAuthenticationToken，而根据用户名加载用户的任务则是交给了 UserDetailsService。

## UserDetailsService

```java
public interface UserDetailsService {
    // 查询用户，去哪里查询自己实现，一般是数据库。
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

需要我们自己编写一个类实现 UserDetailsService 接口，在数据库中查询出登录的用户，将其包装成 UserDetails 对象返回。

这个接口的实现类写完需要在 Spring Security 配置中注册我们自己写的实现类，否则 Spring Security 是不知道你实现了这个接口的。

## UserDetails

```java
public interface UserDetails extends Serializable {
    // 当前认证用户的权限列表
    Collection<? extends GrantedAuthority> getAuthorities();
    // 当前认证用户密码
    String getPassword();
    // 当前认证用户用户名
    String getUsername();
    // 账户是否没过期
    boolean isAccountNonExpired();
    // 账户是否没被锁定（可以用来做黑名单）
    boolean isAccountNonLocked();
    // 账户凭据是否没过期
    boolean isCredentialsNonExpired();
    // 账户是否启用
    boolean isEnabled();
}
```

是一个接口，用来包装对应的数据库查出来的我们自己的 User 对象，包装完之后交给 Spring Security 去判断当前认证的用户账号情况。

## DaoAuthenticationProvider 比对密码

以下代码从源码中来，又跟源码不大一样，为了好看一些，有做删减。

**先从数据库加载用户：**

```java
protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
    try {
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        if (loadedUser == null) {
            // 找不到直接报错，
        } else {
            return loadedUser;
        }
    } catch (UsernameNotFoundException | InternalAuthenticationServiceException | Exception e) {
        // 异常处理
    }
}
```

**比对密码：**

```java
protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
    if (authentication.getCredentials() == null) {
        // 前台没传密码，直接报错
    } else {
        String presentedPassword = authentication.getCredentials().toString();
        // 比对密码，passwordEncoder 是需要注册的密码器。
        if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            // 密码不匹配，直接报错
        }
    }
}
```

**PasswordEncoder：**是一个接口，用来定义一些密码器的，Spring Security 提供了多种不同的密码策略，需要选择一种密码策略，在配置文件中注册。通常使用 BCryptPasswordEncoder。

{% note default %}
**小结：**
{% endnote %}

1. 顶层接口，一切的开始：AuthenticationManager，用于管理认证的每一个环节。
2. Authentication：认证对象，所有的一切都是为它服务的，放在 SecurityContext 中传来传去的。每一个认证环节的开始和结束都是它。
3. ProviderManager：AuthenticationManager 最重要的实现类，那也就是最重要的一个认证环节。
4. ProviderManager 把自己的认证工作委托给了多个 AuthenticationProvider，只要有一个认证成功了就认为是成功的。
5. AuthenticationManager 和 ProviderManager 的职责是不同的，主要就是它们的认证策略不同。
6. 具体的登录密码比对工作交给了 DaoAuthenticationProvider，这也是 ProviderManager 最重要的一个 AuthenticationProvider。
7. 通常需要实现 UserDetailsService 和 UserDetails，DaoAuthenticationProvider 根据这两个对象来校验密码。

写这篇文章的目的主要是学习 Spring Security 的过程中有很多的不理解的地方，很多类名都特别长，长的又差不多，感觉很难理解。所以想要写一篇文章来帮助自己巩固这部分的知识，这篇文章的一些内容也是在网上其他文章中出现过的。

推荐一个 GitHub 上的一个 Spring Security 的仓库，写的真的不错：[向大佬学习](https://github.com/rookie-ricardo/spring-boot-learning-demo)

---

本来鉴权部分应该是要单独的写一篇文章来总结的，但是看了看 Spring Security 的动态鉴权，看是看的明白个大概，就是道行太浅还用不上这么高级的东西。这里就先简单的介绍一下简单的鉴权方案吧。

有需要动态鉴权的同学也可以先看看这两篇文章，顺带一提，大佬写的文章真的不错。

- [Spring Security 鉴权流程](https://zhuanlan.zhihu.com/p/365515214)
- [SpringSecurity动态鉴权流程解析](https://juejin.cn/post/6847902222668431368)

**鉴权策略：**

1. 在 SecurityConfig 配置文件中手动开启 `@EnableGlobalMethodSecurity(prePostEnabled = true)`
2. 在 UserDetailService.loadUserByUsername 方法中查询出用户所具有的全部权限
3. 将权限封装在 UserDetail.getAuthorities() 方法中供 Spring Security 获取
4. 在每个需要授权的 API 上加上 `@PreAuthorize("hasAuthority('权限值')")` 注解
5. 如果用户权限列表有该接口对应权限值就能访问对应接口

---

> **前人栽树：**
> 
> - [Spring Security 认证流程](https://zhuanlan.zhihu.com/p/365513384)
> - [SpringSecurity+JWT认证流程解析](https://juejin.cn/post/6846687598442708999)