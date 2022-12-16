---
title: Java并发：volatile
abbrlink: 63338
description: volatile 读：直接从主内存读；volatile 写：立即刷新到主内存。
date: 2022-08-25 15:26:42
categories:
  - Java
  - 并发
---

# volatile 的特性

当我们声明共享变量为 volatile 后，对这个变量的读/写将会很特别。理解 volatile特性的一个好方法是把对 volatile 变量的单个读/写，看成是使用同一个锁对这些单个读/写操作做了同步。下面我们通过具体的示例来说明，请看下面的示例代码：

```java
class VolatileFeaturesExample {
    volatile long vl = 0L; //使用 volatile 声明 64 位的 long 型变量
    public void set(long l) {
        vl = l; //单个 volatile 变量的写
    }
    public void getAndIncrement () {
        vl++; //复合（多个）volatile 变量的读/写
    }
    public long get() {
        return vl; //单个 volatile 变量的读
    }
}
```

假设有多个线程分别调用上面程序的三个方法，这个程序在语义上和下面程序等价：

```java
class VolatileFeaturesExample {
    long vl = 0L; // 64 位的 long 型普通变量
    public synchronized void set(long l) { //对单个的普通变量的写用同一个锁同步
        vl = l;
    }
    public void getAndIncrement () { //普通方法调用
        long temp = get(); //调用已同步的读方法
        temp += 1L; //普通写操作
        set(temp); //调用已同步的写方法
    }
    public synchronized long get() { //对单个的普通变量的读用同一个锁同步
        return vl;
    }
}
```

如上面示例程序所示，对一个 volatile 变量的单个读/写操作，与对一个普通变量的读/写操作使用同一个锁来同步，它们之间的执行效果相同。

锁的 happens-before 规则保证释放锁和获取锁的两个线程之间的内存可见性，这意味着对一个 volatile 变量的读，总是能看到（任意线程）对这个 volatile 变量最后的写入。

锁的语义决定了临界区代码的执行具有原子性。这意味着即使是 64 位的 long 型和 double 型变量，只要它是 volatile 变量，对该变量的读写就将具有原子性。如果是多个 volatile 操作或类似于 volatile++ 这种复合操作，这些操作整体上不具有原子性。

简而言之，volatile 变量自身具有下列特性：

- 可见性。对一个 volatile 变量的读，总是能看到（任意线程）对这个 volatile 变量最后的写入。
- 原子性：对任意单个 volatile 变量的读/写具有原子性，但类似于 volatile++ 这种复合操作不具有原子性。

# volatile 写-读建立的 happens before 关系

上面讲的是 volatile 变量自身的特性，对程序员来说，volatile 对线程的内存可见性的影响比 volatile 自身的特性更为重要，也更需要我们去关注。

从内存语义的角度来说，volatile 的写-读与锁的释放-获取有相同的内存效果：volatile 写和锁的释放有相同的内存语义；volatile 读与锁的获取有相同的内存语义。

请看下面使用 volatile 变量的示例代码：

```java
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
    public void writer() {
        a = 1;          //1
        flag = true;    //2
    }
    public void reader() {
        if (flag) {     //3
            int i = a;  //4
            ......
        }
    }
}
```

假设线程 A 执行 writer()方法之后，线程 B 执行 reader()方法。根据 happens before 规则，这个过程建立的 happens before 关系可以分为两类：

1. 根据程序次序规则，1 happens before 2; 3 happens before 4。
2. 根据 volatile 规则，2 happens before 3。
3. 根据 happens before 的传递性规则，1 happens before 4。

上述 happens before 关系的图形化表现形式如下：

![volatile-1](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/volatile-1.png)

在上图中，每一个箭头链接的两个节点，代表了一个 happens before 关系。黑色箭头表示程序顺序规则；橙色箭头表示 volatile 规则；蓝色箭头表示组合这些规则后提供的 happens before 保证。

这里 A 线程写一个 volatile 变量后，B 线程读同一个 volatile 变量。A 线程在写volatile 变量之前所有可见的共享变量，在 B 线程读同一个 volatile 变量后，将立即变得对 B 线程可见。

# volatile 写的内存语义

> 当写一个 volatile 变量时，JMM 会把该线程对应的本地内存中的共享变量值刷新到主内存。

以上面示例程序 VolatileExample 为例，假设线程 A 首先执行 writer()方法，随后线程 B 执行 reader()方法，初始时两个线程的本地内存中的 flag 和 a 都是初始状态。下图是线程 A 执行 volatile 写后，共享变量的状态示意图：

![volatile-2](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/volatile-2.png)

如上图所示，线程 A 在写 flag 变量后，本地内存 A 中被线程 A 更新过的两个共享变量的值被刷新到主内存中。此时，本地内存 A 和主内存中的共享变量的值是一致的。

# volatile 读的内存语义

> 当读一个 volatile 变量时，JMM 会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

当读一个 volatile 变量时，JMM 会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

下面是线程 B 读同一个 volatile 变量后，共享变量的状态示意图：

![volatile-3](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/volatile-3.png)

如上图所示，在读 flag 变量后，本地内存 B 包含的值已经被置为无效。此时，线程 B 必须从主内存中读取共享变量。线程 B 的读取操作将导致本地内存 B 与主内存中的共享变量的值也变成一致的了。

如果我们把 volatile 写和 volatile 读这两个步骤综合起来看的话，在读线程 B 读一个 volatile 变量后，写线程 A 在写这个 volatile 变量之前所有可见的共享变量的值都将立即变得对读线程 B 可见。

{% note default %}
**volatile 读写内存语义总结：**
{% endnote %}

- 线程 A 写一个 volatile 变量，实质上是线程 A 向接下来将要读这个 volatile 变量的某个线程发出了（其对共享变量所在修改的）消息。
- 线程 B 读一个 volatile 变量，实质上是线程 B 接收了之前某个线程发出的（在写这个 volatile 变量之前对共享变量所做修改的）消息。
- 线程 A 写一个 volatile 变量，随后线程 B 读这个 volatile 变量，这个过程实质上是线程 A 通过主内存向线程 B 发送消息。

---

> **巨人的肩膀：**
> 
> - [深入理解Java内存模型](https://www.infoq.cn/minibook/java_memory_model)