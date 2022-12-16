---
title: Java并发：AQS
abbrlink: 38839
description: Java 并发之魂：AQS。地位不用多说，各种并发同步工具背后都是基于 AQS 实现的，比如 CountDownLatch、ReentrantLock、Semaphore、CyclicBarrier 等。
date: 2022-08-06 21:54:00
categories:
  - Java
  - 并发
---

AQS 的全称是：AbstractQueuedSynchronizer（抽象队列同步器）j.u.c 包下所提供的一系列同步器，也是在 AQS 的帮助下实现的，比如 CountDownLatch、ReentrantLock、Semaphore、CyclicBarrier 等。

同步器都会有一个内部类继承自 AQS 并且重写 AQS 中的一些方法以定制化一些更具特征的功能，一般这个内部类被命名为 Sync。以 Sync 为工具对外提供一些操纵的接口，以此实现自定义同步器。

同步器都拥有让线程陷入阻塞以及唤醒阻塞线程继续执行的能力，之间的区别只是阻塞、唤醒线程的时机、方法、形式不同而已。也正是这些同步器提供了 Java 强大的并发能力。

所以其实这些同步器都必须具备共同的能力：**阻塞、唤醒线程的能力。这项能力由 LockSupport 类提供。**

# LockSupport 的 park 和 unpark

LockSupport 类位于 `java.util.concurrent.locks` 包下，这个类中包含了 park 和 unpark 方法。park 方法可以阻塞当前线程一直到 unpark 方法被调用，unpark 方法也可以提前被调用。但是 unpark 方法是没有被计数的，也就是说提前调用多次 unpark 只会解除后续的一次 park 操作。另外 LockSupport 类是作用在线程上而不是同步器上的，一个线程在新的同步器上调用 park 操作可能会直接返回，因为在此之前可能还有剩余的 unpark 操作。

**使用示例，先 park 再 unpark**

```java
public static void main(String[] args) throws InterruptedException {

    Thread t1 = new Thread(() -> {
        System.out.println("卧槽，我阻塞了。。。");
        LockSupport.park();
        System.out.println("还好醒过来了。。吓死我了md");
    });

    t1.start();
    Thread.sleep(1); // 让 t1 跑一会
    System.out.println("没事，我来将你唤醒。");
    LockSupport.unpark(t1);
}
```

**使用示例，先 unpark 再 park**

```java
public static void main(String[] args) {
    System.out.println("这次我先将你唤醒！");
    LockSupport.unpark(Thread.currentThread());

    System.out.println("先唤醒再阻塞。");
    LockSupport.park();

    System.out.println("醒过来了。。");
}
```

**LockSupport 类中其他阻塞线程的方法：**

- void park(Object blocker)
- void parkNanos(long nanos)
- void parkNanos(Object blocker, long nanos)
- void parkUntil(long deadline)
- void parkUntil(Object blocker, long deadline)
- blocker：记录导致线程阻塞的对象，方便故障排查
- nanos：阻塞当前线程，最长不超过 nanos 纳秒，或者被 unpark 唤醒
- deadline：阻塞当前线程直到 deadline 之前，或者被 unpark 唤醒

# AQS 概述

开头提到的同步器都必须实现以下两个功能：

- 同步资源的管理（例如：CountDownLatch 的倒数、锁的获取和释放），以及同步资源的更新和检查操作。
- 至少有一个方法导致调用线程在同步状态已经被获取时阻塞，以及在其他线程改变这个同步状态时解除阻塞等。

实现这两个功能需要两个操作：acquire（申请同步资源）、release（释放同步资源），AQS 提供了这两个方法。

- **acquire：**尝试申请同步资源，失败阻塞调用的线程，直到同步资源允许其继续执行。例如：ReentrantLock.lock()、CountDownLatch.await() 等方法
- **release：**释放当前线程所持有的同步资源，使得一或多个被 acquire 阻塞的线程继续执行。例如：ReentrantLock.unlock()、CountDownLatch.countDown() 等方法。

AQS 并没有对自定义同步器的同步方法做统一的定义，因此在不同的同步器中 acquire 和 release 的名称可能有所不同，例如：Lock.lock、Semaphore.acquire、CountDownLatch.await、FutureTask.get 他们都是 acquire 方法的体现，而比如 Lock.unlock 则是 release 的体现了。

AQS 背后的基本思想其实很简单，acquire 操作如下：

```java
while (拿不到同步资源) {
    进入阻塞队列排队;
    或者不排队，直接返回;
}
拿到同步资源了，退出排队;
```

release 操作如下：

```java
释放同步资源;
if (同步资源充足，允许被阻塞的线程获取)
    唤醒能获取同步资源的线程;
```

为了实现 acquire 和 release 操作，需要以下的三个基本组件的相互协作：

- 同步状态的原子性管理
- 线程的阻塞与唤醒被阻塞的线程
- 线程排队的队列管理

同时实现这三个功能是可以做到的，但是无法应对各种各样的同步需求，比如在互斥锁中同一时刻只允许有一个线程持有锁，而共享锁则允许同一时刻有多个线程持有锁、以及各种同步器之间的特性无法同时实现。AQS 实际上是将这些组件共同的部分（例如：acquire 和 release）提取出来了，而其他的同步器继承 AQS 来做一些个性化实现。

> 这也是 Java 继承的一种应用，将公共部分提取出来作为父类，再由子类继承父类去做一些个性化定制。只是在自定义同步器中对 AQS 的继承跟常规继承还有点区别。这里的继承更像是一种组合的操作，而不是对 AQS 的扩展。

# 同步资源

**概述：**同步资源表示的是当前线程是否满足继续执行条件。这个条件更像是一个许可证，拿到执行许可证才能继续执行。所以你可以把同步资源理解为执行许可证。

**表示：**AQS 使用 state 属性（int 32位）来设置同步资源，并暴露出 `getState`、`setState` 以及 `compareAndSet` 操作来读取和更新这个属性，这些方法都依赖于 j.u.c.atomic 包的支持。

AQS 并不维护同步资源的值，仅为其提供维护方法，具体如何对同步资源调配将有继承自 AQS 的同步器自行处理。同步器仅需实现对同步资源的获取与释放方法即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等）AQS 在顶层已经实现好了，同步器主要实现也仅能实现以下的几种方法：

- **isHeldExclusively()：**该线程是否正在独占资源。只有用到condition才需要去实现它。
- **tryAcquire(int)：**独占方式。尝试获取资源，成功则返回true，失败则返回false。
- **tryRelease(int)：**独占方式。尝试释放资源，成功则返回true，失败则返回false。
- **tryAcquireShared(int)：**共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- **tryReleaseShared(int)：**共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现 tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared 中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如 ReentrantReadWriteLock。

# 阻塞队列

整个框架的关键就是如何管理被阻塞的线程的队列，该队列是严格的 FIFO 队列，因此，框架不支持基于优先级的同步。

其实这里称之为队列我认为不是很准确，从数据结构的角度它更像是一个双端链表，可能从 FIFO 的特性来说它才更像是一个队列吧。

既然是双向链表，那么就只需要关注其节点的数据结构以及如何组织节点即可。

先看一下节点的数据结构，至于如何组织节点将在分析 acquire、release 的时候做具体介绍，其实也就是介绍节点的创建时机、以及怎样入队出队的时机。

```java
static final class Node {
    // 标识节点当前是共享模式还是独占模式
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    // waitStatus 字段的值，下面会介绍
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    // 线程等待状态，取值在上面
    volatile int waitStatus;
    // 前驱节点的引用
    volatile Node prev;
    // 后继节点的引用
    volatile Node next;
    // 这个就是线程本尊
    volatile Thread thread;
}
```

这个 Node 节点类很明显就是一个标准双端链表的节点类，有 prev、next 指向前驱和后继节点，thread 当然就是具体的线程，waitStatus 当然就是线程在队列中的状态了。

> - **CANCELLED：**表示当前结点已取消调度。当 timeout 或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的结点将不会再变化。
> - **SIGNAL：**表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为 SIGNAL。
> - **CONDITION：**表示结点等待在 Condition 上，当其他线程调用了 Condition 的 signal() 方法后，CONDITION 状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
> - **PROPAGATE：**共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
> - **0：**新结点入队时的默认状态。

注意：负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用 `>0`、`<0` 来判断结点的状态是否正常。

# AQS 结构

```java
// 头结点，你直接把它当做 当前持有锁的线程 可能是最好理解的
private transient volatile Node head;

// 阻塞的尾节点，每个新的节点进来，都插入到最后，也就形成了一个链表
private transient volatile Node tail;

// 同步资源状态，这个字段将由继承 AQS 的自定义同步器维护
private volatile int state;

// 代表当前持有独占锁的线程，继承自 AbstractOwnableSynchronizer
private transient Thread exclusiveOwnerThread;
```

整个 AQS 维护了一个 state（同步资源）和一个阻塞队列，当多线程竞争同步资源被阻塞的线程会进入此队列排队。他们之间的关系如下：

![AQS-1](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/AQS-1.png)

---

> **小结：**
> 
> AQS 是被抽象出来作为各个同步器最重要的工具 Sync 类的父类而存在的。AQS 定义了同步资源 state 属性，并且为其提供维护方法。同步器通过维护同步资源来调用 AQS 中对线程的阻塞入队/唤醒出队等操作。

关于 AQS 的概念性介绍就到这里，关于最重要的 acquire 和 release 方法会再单独开一篇文章做详解。

---

> **巨人的肩膀：**
> 
> - [《The java.util.concurrent Synchronizer Framework》 JUC同步器框架（AQS框架）原文翻译](https://www.cnblogs.com/dennyzhangdd/p/7218510.html)
> - [一行一行源码分析清楚AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer)
> - [Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)