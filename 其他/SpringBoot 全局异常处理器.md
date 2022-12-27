---
title: 【Spring Boot】全局异常处理器
abbrlink: 19537
date: 2021-08-11 16:13:13
hide: true
categories:
 - 代码段记录
---

记录一下最近在开发中遇到的一个问题有以及一个不错的解决方案。

通常情况下一个请求从前端发起到后端的 Controller 接收这个请求，再去调用对应的 Service 方法去处理对应的逻辑，再由 Controller 将处理的结果封装成一个 JSON 对象返回给前端。这是我们的一个正常的请求处理的过程，但是这其中也有一些例外的地方，如果在 Service 处理的时候出现了一些业务上的逻辑问题流程已经无法再继续往下面去走了，这个时候需要在 Service 直接返回到前端需要怎么做？

<!-- more -->

其实可以通过约定不同的返回值给到 Controller 去判断需要怎么处理再怎么返回给前端。但是这样会带来两个问题：
1. 一旦项目中的返回类型多了起来就会造成 Controller 层代码的冗余。
2. Service 一旦返回事务就会提交，这样子就没办法灵活的来控制我们的事务了。当然这也可以通过 Service 层代码编写的逻辑来解决，但这么处理就不是那么的优雅了。

一个优雅的处理方式应该是定义好一个业务异常类 BizException，这个类需要继承 RuntimeException 类，一旦在 Service 中需要返回的时候就抛出 BizException 异常，这样事务就可以回滚，然后再由 Spring Boot 的全局异常处理器来捕获这个异常直接将我们自定义的异常信息返回给前端。

**代码：**

```java
import lombok.Getter;

/**
 * 自定义业务异常
 */
@Getter
public class BizException extends RuntimeException {

    // 此处的异常信息仅仅为了演示而定义的只有一个 errMsg 字段
    // 实际使用中可以根据需要将这个异常信息定义的丰富一些，比如使用一个 Enum 来维护这些异常信息
    private final String errMsg;

    public BizException(String errMsg) {
        super();
        this.errMsg = errMsg;
    }
}

```

```java
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

/**
 * 全局异常处理器
 */
@ControllerAdvice
public class GlobalExceptionHandler {

    // ApiResponse 类为接口全局响应类

    @ResponseBody
    @ExceptionHandler(BizException.class)
    public ApiResponse<String> businessExceptionHandler(BizException e) {
        return ApiResponse.fail(e.getErrMsg());
    }
}
```