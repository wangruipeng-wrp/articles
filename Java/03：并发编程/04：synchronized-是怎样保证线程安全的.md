---
title: synchronized 是怎样保证线程安全的
abbrlink: 11150
date: 2022-11-01 21:00:34
tags:
---

**一句话说明 synchronized 关键字的作用：**
> 保证在 **「同一时刻」** 最多只有 **「一个」** 线程执行该段代码，以达到保证 **「并发安全」** 的效果

上一篇文章讲述的线程不安全的例子，现在就给出解决办法：synchronized。

# synchronized 关键字的使用方式

为了阅读方便，把前一篇文章中的示例代码再放到这里：

```java
public class Task implements Runnable {

    // 共享变量
    static int num = 0;

    @Override
    public void run() {
        // 对共享变量的修改
        for (int i = 0; i < 100_000; i++) {
            num++;
        }
    }

    public static void main(String[] args) {
        final Thread t1 = new Thread(new Task());
        final Thread t2 = new Thread(new Task());
        t1.start();
        t2.start();
        while (t1.isAlive() || t2.isAlive()) { }
        System.out.println("num = " + num);
    }
}
```

可以理解为是一个锁的作用，用来保护需要同步执行的代码，只有拿到锁的线程才能执行被保护的代码。如果使用 synchronized 来保护 `num++` 这行代码 ，那就可以保证 num 变量的线程安全。因为这样同一个时刻就只有一个线程能执行 `num++` 了。被保护代码也称为临界区代码。

synchronized 既然是一把锁，那肯定就需要一个对象来充当这个锁对象，这个对象可以是类对象或者是实例对象。由此就有了两个锁的概念：类锁、对象锁。

再根据使用方式的不同，类锁可以分为：
- 静态方法加锁
- 代码块直接指定类锁

对象锁可以分为：
- 普通方法加锁
- 代码块直接指定对象锁

由于 Java 中的类是全局唯一的对象，而对象则可以存在多个。于是，类锁是全局唯一的，被类锁保护的代码在同一时刻肯定只有唯一一个线程能执行该代码；对象锁在全局可以同时存在多个，被对象锁保护的代码在同一时刻可能会被多个线程同时执行，但这些线程肯定持有不同的对象锁。

## 类锁

由于类锁在全局的唯一性，所以使用类锁保护的代码可以保证在整个 JVM 实例中同一时刻只有一个线程能执行该方法。

### 静态方法加锁

```java
// 将 synchronized 关键字添加在静态方法签名上，以 Task.class 类对象为锁
public static synchronized void incr() {
    for (int i = 0; i < 100; i++) {
        num++;
    }
}
```

### 代码块指定类锁

```java
@Override
public void run() {
    for (int i = 0; i < 100; i++) {
        synchronized (Task.class) {
            num++;
        }
    }
}
```

## 对象锁

被 new 出来或者反射加载的对象，在内存中可以存在多份。所以被对象锁保护的代码同一时刻可能会被多个线程执行，但是这些线程持有的实例对象肯定不同。

### 普通方法加锁

```java
// 将 synchronized 关键字添加在方法签名上，以 this 对象为锁
@Override
public synchronized void run() {
    for (int i = 0; i < 100; i++) {
        num++;
    }
}
```

### 代码块指定对象锁

```java
@Override
public void run() {
    for (int i = 0; i < 100; i++) {
        synchronized (this) {
            num++;
        }
    }
}
```

## 类锁和对象锁的使用场景

类锁的经典使用场景有单例模式的双重检查锁（DCL）实现，由于类锁限制的范围太广泛了，在整个 JVM 实例同一时刻只有一个线程能执行被保护代码，也就是完全放弃了并行带来的性能提升，在使用类锁的时候这一点是需要认真考量的。DCL 的外层判断也是为了降低锁的粒度，基于性能的考量而加的。

对象锁的使用场景就比较多了，早期的并发容器当中使用的都是对象锁来保护并发安全，例如 Vector 和 Hashtable。早期并发容器中对象锁的使用也可以做为一个参考，一个对象内部的资源需要被保护时，可以以这个对象为锁来使用 synchronized 关键字。