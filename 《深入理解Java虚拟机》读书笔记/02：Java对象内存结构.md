---
title: Java对象内存结构
abbrlink: 4996
date: 2022-11-03 18:17:11
---

# 对象的创建

当 new 一个新的对象时，首先检查这个对象所属的类是否已经被加载、解析和初始化。如果没有那必须先执行类加载过程。接下来虚拟机将为新生对象分配内存，为对象分配空间的任务实际上便等同于把一块确定大小的内存块（类加载完毕后对象大小也被确定）从 Java 堆中划分出来。

## 对象内存空间分配

分配空间有两种方式：指针碰撞和空闲列表。

**指针碰撞：**假设Java堆中内存是绝对规整的，所有被使用过的内存都被放在一边，空闲的内存被放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间方向挪动一段与对象大小相等的距离，这种分配方式称为“指针碰撞”（Bump The Pointer）。

**空闲列表：**但如果Java堆中的内存并不是规整的，已被使用的内存和空闲的内存相互交错在一起，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“空闲列表”（Free List）。

> 选择哪种分配方式由Java堆是否规整决定，而Java堆是否规整又由所采用的垃圾收集器是否带有空间压缩整理（Compact）的能力决定。因此，当使用Serial、ParNew等带压缩整理过程的收集器时，系统采用的分配算法是指针碰撞，既简单又高效；而当使用CMS这种基于清（Sweep）算法的收集器时，“理论上”就只能采用较为复杂的空闲列表来分配内存。*（CMS的实现里面，为了能在多数情况下分配得更快，设计了一个叫作Linear Allocation Buffer的分配缓冲区，通过空闲列表拿到一大块分配缓冲区之后，在它里面仍然可以使用指针碰撞方式来分配。）*

## 指向新对象

对象创建在虚拟机中是非常频繁的行为，即使仅仅修改一个指针所指向的位置，在并发情况下也并不是线程安全的，可能出现正在给对象A分配内存，指针还没来得及修改，对象B又同时使用了原来的指针来分配内存的情况。解决这个问题关键是为对象内存分配这个动作提供原子性保证，有两种可选方案：CAS和线程隔离。

**CAS：**对分配内存空间的动作进行同步处理，CAS配上失败重试的方式保证更新操作的原子性

**线程隔离：**把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB），哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完了，分配新的缓存区时才需要同步锁定。是否使用TLAB，可以通过-XX：+/-UseTLAB参数来设定。

## 初始化新对象

**内存空间初始化：**内存分配完成之后，虚拟机必须将分配到的内存空间（但不包括对象头）都初始化为零值，如果使用了TLAB的话，这一项工作也可以ᨀ前至TLAB分配时顺便进行。这步操作保证了对象的实例字段在Java代码中可以不赋初始值就直接使用，使程序能访问到这些字段的数据类型所对应的零值。

**对象头初始化：**接下来，Java虚拟机还要对对象进行必要的设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码（懒加载）、对象的GC分代年龄等信息。这些信息存放在对象的对象头（Object Header）之中。根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

# 堆内存中的对象

对象在堆内存中的存储布局可以划分为三个部分：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

## 对象头

HotSpot虚拟机对象的对象头部分包括两类信息：一类是用于存储对象自身的运行时数据，另一类是类型指针。

**对象自身的运行时数据：**哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。

**类型指针：**即对象指向它的类型元数据的指针，Java虚拟机通过这个指针来确定该对象是哪个类的实例。

此外，如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据。

## 实例数据

实例数据部分是对象真正存储的有效信息，即我们在程序代码里面所定义的各种类型的字段内容，无论是从父类继承下来的，还是在子类中定义的字段都必须记录起来。

HotSpot虚拟机默认的分配顺序为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers，oops）

## 对齐填充

对象的第三部分是对齐填充，这并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。由于HotSpot虚拟机的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说就是任何对象的大小都必须是8字节的整数倍。对象头部分已经被精心设计成正好是8字节的倍数（1倍或者2倍），因此，如果对象实例数据部分没有对齐的话，就需要通过对齐填充来补全。

# 对象的访问定位

Java程序会通过栈上的reference数据来操作堆上的具体对象，主流的访问方式有两种：句柄、直接指针。

**句柄：**使用句柄访问的话，Java堆中将可能会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自具体的地址信息。

使用句柄来访问的最大好处就是reference中存储的是稳定句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要被修改。

![通过句柄访问对象](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/通过句柄访问对象.png)

**直接指针：**使用直接指针访问的话，Java堆中对象的内存布局就必须考虑如何放置访问类型数据的相关信息，reference中存储的直接就是对象地址，如果只是访问对象本身的话，就不需要多一次间接访问的开销。

使用直接指针来访问最大的好处就是速度更快，它节省了一次指针定位的时间开销，由于对象访问在Java中非常频繁，因此这类开销积少成多也是一项极为可观的执行成本。

![通过直接指针访问对象](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/通过直接指针访问对象.png)

---

> **巨人的肩膀：**
> - 《深入理解Java虚拟机》