---
title: Java并发：内存模型（JMM）
abbrlink: 41498
description: 并发问题产生的三大根源是「原子性」「可见性」「有序性」，引入 Java 内存模型就是为了解决这三个问题。
date: 2022-07-27 00:49:31
categories:
  - Java
  - 并发
---

并发问题产生的三大根源是「原子性」「可见性」「有序性」，引入 Java 内存模型就是为了解决这三个问题。

# 并发问题是怎么产生的

{% note default %}
**为什么需要多线程？**
{% endnote %}

众所周知，CPU、内存、I/O 设备的速度是有极大差异的，为了合理利用 CPU 的高性能，平衡这三者的速度差异，计算机体系结构、操作系统、编译程序都做出了贡献，主要体现为：

- 操作系统增加了进程、线程，以分时复用 CPU，进而均衡 CPU 与 I/O 设备的速度差异；// 导致 `原子性` 问题 
- CPU 增加了缓存，以均衡与内存的速度差异；// 导致 `可见性` 问题
- 编译程序优化指令执行次序，使得缓存能够得到更加合理地利用；// 导致 `有序性` 问题

## 原子性是怎么产生并发问题的

> 原子性问题由分时复用引起

原子性：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

```java
int i = 1;

// 线程1执行
i += 1;

// 线程2执行
i += 1;
```

这里需要注意的是：`i += 1` 需要三条 CPU 指令 
1. 将变量 i 从内存读取到 CPU寄存器； 
2. 在CPU寄存器中执行 i + 1 操作；
3. 将最后的结果i写入内存（缓存机制导致可能写入的是 CPU 缓存而不是内存）。

由于CPU分时复用（线程切换）的存在，线程1执行了第一条指令后，就切换到线程2执行，假如线程2执行了这三条指令后，再切换会线程1执行后续两条指令，将造成最后写到内存中的i值是2而不是3。

## 可见性是怎么产生并发问题的

> 可见性问题由 CPU 缓存引起

可见性：一个线程对共享变量的修改，另外一个线程能够立刻看到。

```java
//线程1执行的代码
int i = 0;
i = 10;
 
//线程2执行的代码
j = i;
```

假若执行线程1的是CPU1，执行线程2的是CPU2。由上面的分析可知，当线程1执行 i =10这句时，会先把i的初始值加载到CPU1的高速缓存中，然后赋值为10，那么在CPU1的高速缓存当中i的值变为10了，却没有立即写入到主存当中。

此时线程2执行 j = i，它会先去主存读取i的值并加载到CPU2的缓存当中，注意此时内存当中i的值还是0，那么就会使得j的值为0，而不是10。

这就是可见性问题，线程1对变量i修改了之后，线程2没有立即看到线程1修改的值。

## 有序性是怎么产生并发问题的

> 有序性问题由重排序引起

有序性：即程序执行的顺序按照代码的先后顺序执行。举个简单的例子，看下面这段代码：

```java
int i = 0;
boolean flag = false;
i = 1;        //语句1  
flag = true;  //语句2
```

上面代码定义了一个int型变量，定义了一个boolean类型变量，然后分别对两个变量进行赋值操作。从代码顺序上看，语句1是在语句2前面的，那么JVM在真正执行这段代码的时候会保证语句1一定会在语句2前面执行吗? 不一定，为什么呢? 这里可能会发生指令重排序（Instruction Reorder）。

在执行程序时为了提高性能，编译器和处理器常常会对指令做重排序。重排序分三种类型：

- 编译器优化的重排序。编译器在不改变单线程程序语义（as-if-serial 语义）的前提下，可以重新安排语句的执行顺序。
- 指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-Level Parallelism， ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
- 内存系统的重排序。由于处理器使用缓存和读 / 写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

从 java 源代码到最终实际执行的指令序列，会分别经历下面三种重排序：

![重排序从源代码到指令序列所经历步骤](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/重排序从源代码到指令序列所经历步骤.png)

上述的 1 属于编译器重排序，2 和 3 属于处理器重排序，这些重排序都可能会导致多线程程序出现内存可见性问题。
- 对于编译器重排序，JMM 的编译器重排序规则会禁止特定类型的编译器重排序。
- 对于处理器重排序，JMM 的处理器重排序规则会要求 java 编译器在生成指令序列时，插入特定类型的内存屏障（memory barriers，intel 称之为 memory fence）指令，通过内存屏障指令来禁止特定类型的处理器重排序。

> 不管是编译器重排序还是处理器重排序都可以帮助提升程序性能，只是这样的重排序在特定的情况下可能会让代码变得不可靠，所以 JMM 要求在特定情况下是禁止重排序的，此处并非一昧禁止重排序。

同样的，JMM 也不是一昧允许重排序，那么：**重排序的规则是什么？**
由于篇幅问题此处不做展开，感兴趣的同学请移步：[重排序所遵循的规则](https://www.wrp.cool/posts/41133/)

# JMM 是怎么解决并发问题的

- **JMM 为我们提供了什么工具来解决并发问题？**
- JMM 为我们提供了 synchronized、final、volatile、Lock锁、Happens-Before 规则。利用好这些工具能帮我们解决并发问题。

## JMM 怎么解决原子性问题

> JMM 要求对基本数据类型的变量的读取和赋值操作是原子性操作，即这些操作是不可被中断的，要么执行，要么不执行。 

请分析以下哪些操作是原子性操作：

```java
x = 10;     // 语句1：直接将数值10赋值给x，也就是说线程执行这个语句的会直接将数值10写入到工作内存中
y = x;      // 语句2：包含2个操作，它先要去读取x的值，再将x的值写入工作内存，虽然读取x的值以及 将x的值写入工作内存 这2个操作都是原子性操作，但是合起来就不是原子性操作了。
x++;        // 语句3：x++包括3个操作：读取x的值，进行加1操作，写入新的值。
x = x + 1;  // 语句4：同语句3
```

上面4个语句只有语句1的操作具备原子性。

也就是说，只有简单的读取、赋值（必须是将数字赋值给某个变量）才是原子操作。变量之间的相互赋值不是原子操作。

> 从上面可以看出，JMM 只保证了基本读取和赋值是原子性操作，如果要实现更大范围操作的原子性，可以通过 synchronized 和 Lock 来实现。由于 synchronized 和 Lock 能够保证任一时刻只有一个线程执行该代码块，那么自然就不存在原子性问题了，从而保证了原子性。

## JMM 怎么解决可见性问题

JMM 提供了 volatile 关键字来保证可见性。

当一个共享变量被 volatile 修饰时，它会保证修改的值会立即被更新到主存，当有其他线程需要读取时，它会去内存中读取新值。

由于篇幅问题此处不对 volatile 做展开，感兴趣的同学请移步[Java并发：volatile](https://www.wrp.cool/posts/63338/)

而普通的共享变量不能保证可见性，因为普通共享变量被修改之后，什么时候被写入主存是不确定的，当其他线程去读取时，此时内存中可能还是原来的旧值，因此无法保证可见性。

> 另外，synchronized 和 Lock 也能够保证可见性，synchronized 和 Lock 能保证同一时刻只有一个线程获取锁然后执行同步代码，并且在释放锁之前会将对变量的修改刷新到主存当中。因此可以保证可见性。

## JMM 怎么解决有序性问题

JMM 提供了 Happens-Before 规则在一定程度上保证了有序性

一个 happens-before 规则通常对应于多个编译器重排序规则和处理器重排序规则。对于 java 程序员来说，happens-before 规则简单易懂，它避免程序员为了理解 JMM 提供的内存可见性保证而去学习复杂的重排序规则以及这些规则的具体实现。 

{% note simple default no-icon %}
**规则一：程序顺序规则**
在一个线程中，按照代码的顺序，前面的操作 Happens-Before 于后面的任意操作。
{% endnote %}

在这条规则下可能会有一个疑问：**既然指令可以重排序又怎么保证程序顺序规则？**

> 以 Happens-Before 的角度回答这个问题：

JMM 通过 Happens-Before 关系向开发人员提供**跨越线程**的内存可见性保证。

如果一个操作的执行结果对另外一个操作可见，那么这两个操作之间必然存在 Happens-Before 管理。

其次，Happens-Before 关系只是描述结果的可见性，并不表示指令执行的先后顺序，也就是说只要不对结果产生影响，仍然允许指令的重排序。

> 以重排序的角度回答这个问题：

在程序顺序规则下，JMM 并不是不允许重排序，JMM 仅仅要求前一个操作（执行的结果）对后一个操作可见。

假设根据程序顺序规则有 A Happens-Before B 如果操作A的结果不需要对操作B可见，并且操作A和操作B重排序前后的执行结果一致；

在这种情况下 JMM 会认为这种重排序并不非法，JMM 允许这种重排序。

{% note simple default no-icon %}
**规则二：volatile变量规则**
对一个 volatile 变量的写操作，Happens-Before 于后续对这个变量的读操作。
{% endnote %}

也就是说对一个 volatile 变量而言，肯定不会发生可见性问题。

因为 volatile 写完会被立即刷回主内存中，而读操作发生在这之后，那么每次读取都会读取到最新值。

{% note simple default no-icon %}
**规则三：传递规则**
如果A Happens-Before B，并且B Happens-Before C，则A Happens-Before C。
{% endnote %}

这个规则比较简单，此处不做展开。

{% note simple default no-icon %}
**规则四：锁定规则**
对一个锁的解锁操作 Happens-Before 于后续对这个锁的加锁操作。
{% endnote %}

这个很好理解，同一把锁的情况下，肯定是要先解锁才能再次上锁。已经锁了的情况下总不能再锁一次吧。

{% note simple default no-icon %}
**规则五：线程启动规则**
如果线程A调用线程B的 start() 方法来启动线程B，则 start() 操作 Happens-Before 于线程B中的任意操作。
{% endnote %}

我们也可以这样理解线程启动规则：线程A启动线程B之后，线程B能够看到线程A在启动线程B之前的操作。

```java
//在线程A中初始化线程B
Thread threadB = new Thread(()->{
    //此处的变量x的值是多少呢？答案是100
});
//线程A在启动线程B之前将共享变量x的值修改为100
x = 100;
//启动线程B
threadB.start();
```

上述代码是在线程A中执行的一个代码片段，根据线程启动规则，线程A启动线程B之后，线程B能够看到线程A在启动线程B之前的操作，在线程B中访问到的x变量的值为100。

{% note simple default no-icon %}
**规则六：线程终结规则**
线程A等待线程B完成（在线程A中调用线程B的 join() 方法实现），当线程B完成后（线程A调用线程B的 join() 方法返回），则线程A能够访问到线程B对共享变量的操作。
{% endnote %}

我们也可以这样理解线程终结规则：线程A的 join() 方法返回之后，线程A能看到线程B的所有操作

```java
private static int x = 0;

public static void main(String[] args) throws InterruptedException {

    Thread t = new Thread(() -> {
        //在线程B中，将共享变量x的值修改为100
        x = 100;
    });
    //在线程A中启动线程B
    t.start();
    //在线程A中等待线程B执行完成
    t.join();
    //此处访问共享变量x的值为100
    System.out.println(x);
}
```

{% note simple default no-icon %}
**规则七：线程中断规则**
对线程 interrupt() 方法的调用 Happens-Before 于被中断线程的代码检测到中断事件的发生。
{% endnote %}

我们也可以这样理解线程中断规则：在 InterruptedException 的 catch 代码块中能够看到调用 interrupt() 方法之前的操作。

```java
//在线程A中将x变量的值初始化为0
private int x = 0;

public void execute(){
    //在线程A中初始化线程B
    Thread threadB = new Thread(()->{
        //线程B检测自己是否被中断
        while (!Thread.currentThread().isInterrupted()){ }
        //如果线程B被中断，则此时X的值为100
        System.out.println(x);
    });
    //在线程A中启动线程B
    threadB.start();
    //在线程A中将共享变量X的值修改为100
    x = 100;
    //在线程A中中断线程B
    threadB.interrupt();
}
```

{% note simple default no-icon %}
**规则八：对象终结规则**
一个对象的初始化完成 Happens-Before 于它的 finalize() 方法的开始。
{% endnote %}

```java
public class TestThread {

   public TestThread(){
       System.out.println("构造方法");
   }

    @Override
    protected void finalize() throws Throwable {
        System.out.println("对象销毁");
    }

    public static void main(String[] args){
        new TestThread();
        System.gc();
    }
}
```

---

> **巨人的肩膀：**
> 
> - [Java 并发 - 理论基础](https://pdai.tech/md/java/thread/java-thread-x-theorty.html)
> - [深入理解Java内存模型](https://www.infoq.cn/minibook/java_memory_model)
> - [何为Happens-Before原则？这次彻底懂了！](https://cloud.tencent.com/developer/article/1734515)