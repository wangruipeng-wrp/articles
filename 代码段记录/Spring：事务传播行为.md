---
title: Spring基础：事务传播行为
abbrlink: 33401
date: 2021-06-13 15:10:39
description: 记录 Spring 7种事务传播行为
categories: 
 - Spring
 - Spring 基础
---

记录一下Spring中的事务传播行为，也就是 `@Transactional` 这个注解中的 `propagation` 属性的值分别是什么，有什么，各自又代表了什么。

事务传播行为就是：<font color=blue>当方法与方法之间有发生嵌套调用的情况下，父级方法和子级方法之间的事务如何处理。</font>通过我们定义不同的传播行为，可以使得父级方法和子级方法有不同的处理事务的方式。

<!-- more -->

Spring 中的7种事务传播行为：

事务传播类型 | 说明 | 举例
-- | -- | -- 
Propagation.REQUIRED | 如果当前没有事务，就新建一个事务，如果已经存在一个事务中，则加入到这个事务中。|父级有子级就父级共享父级没有子级就自己创建。
Propagation.SUPPORTS | 支持当前事务，如果当前没有事务，就以非事务的方式运行。 |父级有子级就父级共享父级没有就一起没有。
Propagation.MANDATORY | 使用当前的事务，如果当前没有事务，就抛出异常。 |父级必须要有。
Propagation.REQUIRES_NEW | 新建事务，如果当前存在事务，把当前事务挂起。 | 子级肯定有父级有父级自己的，子级不父级共享。
Propagation.NOT_SUPPORTED | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 | 子级肯定没有，就父级有子级也不要。
Propagation.NEVER | 以非事务方式执行，如果当前存在事务，则抛出异常。 | 子级没有父级也不能有。
Propagation.NESTED | 如果当前存在事务，则成为当前事务的子事务。如果当前没有事务，则执行与`Propagation.REQUIRED`类似的操作。 |父级有子级就父级共享，当子级回滚时也不影父级和其他兄弟。但是如父级回滚了，那子级肯定跟父级一起回滚。

> 参考文章：
> [Spring事务传播行为详解](https://segmentfault.com/a/1190000013341344)