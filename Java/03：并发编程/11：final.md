---
title: Java并发：final
abbrlink: 38583
date: 2022-08-04 21:50:40
description: final 是一个在 Java 基础阶段就会学到的关键字，首先回顾一下 final 的基本用法，再进一步深入了解 final 在并发中的应用。
categories:
  - Java
  - 并发
---

final 是一个在 Java 基础阶段就会学到的关键字，首先回顾一下 final 的基本用法，再进一步深入了解 final 在并发中的应用。

# final 基本用法

## 修饰类

当某个类的整体定义为 final 时，就表明了你不能打算继承该类，而且也不允许别人这么做。即这个类是不能有子类的。

注意：final 类中的所有方法都隐式为 final，因为无法覆盖他们，所以在 final 类中给任何方法添加 final 关键字是没有任何意义的。

{% note default %}
**那么怎么扩展 final 类呢？**
{% endnote %}

设计模式中最重要的两种关系，一种是继承/实现；另外一种是组合关系。所以当遇到不能用继承的（final 修饰的类），应该考虑用组合，如下代码大概写个组合实现的意思：

```java
class MyString{

    private String innerString;

    // ...init & other methods

    // 支持老的方法
    public int length(){
        return innerString.length(); // 通过innerString调用老的方法
    }

    // 添加新方法
    public String toMyString(){
        //...
    }
}
```

## 修饰方法

final 修饰的方法不能被重写，不过重写的前提是继承，如果 final 修饰的方法同时又是 private 的将会导致子类无法重写此方法，子类可以定义相同的方法名和参数，该方法将成为子类的新方法，而不是继承自父类的方法。

final 方法虽然不能被重写，但是可以被重载的，如下代码是正确的：

```java
public class FinalDemo {
    public final void finalFunction() {

    }
    public final void finalFunction(String name) {

    }
}
```

类中所有的 private 方法都是隐式 final 的，由于子类看不到父类的 private 方法，所以也就不能重写它。在代码中仍然能够对 private 方法添加 final 修饰符，但是这并没有什么用处。

```java
class A {
    private final void func() {
        System.out.println("func from A");
    }
}

class B extends A {
    public void func() {
        System.out.println("func from B");
    }
}

public class Demo {
    public static void main(String[] args) {
        A a = new B();
        a.func(); // 编译错误
    }
}
```

## 修饰参数

final 修饰的参数不能被改变，被 final 修饰的基本数据类型的参数其值不能被改变，被 final 修饰的引用数据类型的参数其地址不能被改变，但对象内部的数据可以被改变。

```java
public void finalDemo(final int i, final User user) {
    user.name = "zhangsan"; // 正确
    i = 10;                 // 错误
    user = new User();      // 错误
}
```

## 修饰变量

**被 final 修饰的属性不一定都是编译器常量**

```java
public class Test {
    //编译期常量
    final int i = 1;
    final static int J = 1;
    final int[] a = {1,2,3,4};

    //非编译期常量
    Random r = new Random();
    final int k = r.nextInt();
}
```

`k` 的值由随机数对象决定，所以不是所有的 final 修饰的字段都是编译期常量，只是 `k` 的值在被初始化后无法被更改。

**static final 变量**

一个既是static又是final 的字段只占据一段不能改变的存储空间，它必须在定义的时候进行赋值，否则编译器将不予通过。

```java
public class Test {
    static Random r = new Random();
    final int k = r.nextInt(10);
    static final int k2 = r.nextInt(10); 
    public static void main(String[] args) {
        Test t1 = new Test();
        System.out.println("k="+t1.k+" k2="+t1.k2);
        Test t2 = new Test();
        System.out.println("k="+t2.k+" k2="+t2.k2);
    }
}
```

上面的代码某次输出结果：

```
k=6 k2=5
k=1 k2=5
```

对于不同的对象`k`的值是不同的，但是`k2`的值却是相同的，这是为什么呢？
因为 static 关键字所修饰的字段并不属于一个对象，而是属于这个类的。也可简单的理解为 static final 所修饰的字段仅占据内存的一个一份空间，一旦被初始化之后便不会被更改。

---

对于 final 的基本用法就介绍到这里，接下来介绍 final 在并发中的应用：

# final 域的重排序规则

1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
2. 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

**写 final 域的重排序规则：**

- JMM 禁止编译器把 final 域的写重排序到构造函数之外。
- 编译器会在 final 域的写之后，构造函数 return 之前，插入一个 StoreStore 屏障。这个屏障禁止处理器把 final 域的写重排序到构造函数之外。

**读 final 域的重排序规则：**

- 在一个线程中，初次读对象引用与初次读该对象包含的 final 域，JMM 禁止处
理器重排序这两个操作（注意，这个规则仅仅针对处理器）。编译器会在读 final 域操作的前面插入一个 LoadLoad 屏障。

> **巨人的肩膀：**
> 
> - [深入理解Java内存模型](https://www.infoq.cn/minibook/java_memory_model)
> - [关键字: final详解](https://pdai.tech/md/java/thread/java-thread-x-key-final.html)