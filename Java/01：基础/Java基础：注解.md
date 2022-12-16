---
title: Java：注解
abbrlink: 28807
date: 2020-09-13 11:11:52
categories:
 - Java
 - 高级特性
---

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface MyAnnotation {
    String value() default "我的注解";

    String name() default "name";
}
```
以上就是Java自定义注解的格式，其中的四个注解就是元注解，也就是我们接下注解部分来的重点。

<!-- more -->

# 元注解
普通的注解就是用来修饰例如变量、方法和类的。而元注解是用来修饰注解的，是修饰注解的注解。下面的四个注解就是元注解。

**@Target**
目标的意思，表示的是注解可以修饰的目标。参数是一个`java.lang.annotation.ElementType`类型的数组。
```java
@Target({
        ElementType.TYPE,               // 类、接口、枚举类
        ElementType.FIELD,              // 成员变量、枚举常量
        ElementType.METHOD,             // 成员方法
        ElementType.PARAMETER,          // 方法参数
        ElementType.CONSTRUCTOR,        // 构造方法
        ElementType.LOCAL_VARIABLE,     // 局部变量
        ElementType.ANNOTATION_TYPE,    // 注解类
        ElementType.PACKAGE,            // 包
        ElementType.TYPE_PARAMETER,     // 类型参数，jdk1.8新增
        ElementType.TYPE_USE            // 使用类型的任何地方，jdk1.8新增
})
@interface MyAnnotation {
}
```

**@Retention**
有点类似于生命周期的意思，表示的是注解在程序中作用的范围，参数是`java.lang.annotation.RetentionPolicy`
```java
@Retention(RetentionPolicy.SOURCE)      // 源文件保留
@Retention(RetentionPolicy.CLASS)       // 编译后保留
@Retention(RetentionPolicy.RUNTIME)     // 运行时保留
@interface MyAnnotation {
}
```

**@Inherited**
Inherited注解的作用是：使被它修饰的注解具有继承性（如果某个类使用了被@Inherited修饰的注解，则其子类将自动具有该注解）。
```java
@Inherited
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyInheritedAnnotation { 
	public String name();
}
```
```java
@MyInheritedAnnotation(name="parent")
public class Parent {
}
```
```java
public class Child extends Parent{
	public static void main(String[] args) {
		Class<Child> child = Child.class;
		MyInheritedAnnotation annotation = child.getAnnotation(MyInheritedAnnotation.class);
		System.out.println(annotation.name());
	}
}
```
运行main方法打印结果：`parent`

**@Documented**
Documented注解的作用是：描述在使用 javadoc 工具为类生成帮助文档时是否要保留其注解信息。

> 以上内容参考：[JAVA核心知识点--元注解详解](https://blog.csdn.net/pengjunlee/article/details/79683621#meta-annotation%EF%BC%88%E5%85%83%E6%B3%A8%E8%A7%A3%EF%BC%89)

**彩蛋：**
将名称定义成value的值在使用时是可以省略value属性不写的。
例如上面定义的`MyAnnotation`注解，在设置值时`value`属性是可以省略属性名称的，而`name`属性则不可以省略属性名称。

`@MyAnnotation("value")` 表示指定了value的值
`@MyAnnotation(name = "name")` 表示指定了name的值
`@MyAnnotation(value = "value", name = "name")` 表示指定了value和name的值，如果指定了多个值则属性名不可以省略