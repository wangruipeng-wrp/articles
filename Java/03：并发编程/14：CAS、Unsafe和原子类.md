---
title: Java并发：Unsafe、CAS和原子类
abbrlink: 61146
date: 2022-08-04 21:50:18
description: Java 中 CAS 由 Unsafe 类实现，原子类是对 CAS 的包装，对外屏蔽了 Unsafe 类，以及提供一些方便的操作。
categories:
  - Java
  - 并发
---

> 我在网上看到一些文章写着：CAS 是一种自旋锁。这是一种错误的观念，CAS 跟自旋其实没什么关系，只是当 CAS 失败的时候通常会使用自旋补偿罢了。换句话说，自旋是 CAS 的一种常见的补偿操作，二者并无直接关系。

我个人认为程序员对于 CAS 这样的一种工具其实不应该是仅仅当作工具去看待，更多的是要掌握 CAS 这种思想，并且运用到实际开发中。比如在商品下单的库存校验部分就可以用到 CAS 思想。

```sql 
update set reserve = reserve - #{buyCount} from goods where reserve >= #{buyCount}
```

由于单条 SQL 执行的时候具有原子性，就算是秒杀的时候也不会导致库存不足的情况出现，这也是 CAS 的一种体现。

这也印证了 CAS 和自旋并无直接关系，此处的 CAS 失败应该直接通知用户库存不足，而不是做自旋等待。

---

Java 中 CAS 由 Unsafe 类实现，原子类是对 CAS 的包装，对外屏蔽了 Unsafe 类，以及提供一些方便的操作。

CAS 的全称为 Compare-And-Swap ，直译就是对比交换。是一条 CPU 的原子指令（cmpxchg 指令），其作用是让 CPU 先进行比较两个值是否相等，然后原子地更新某个位置的值。就是说 CAS 是靠硬件实现的，JVM 只是封装了汇编调用，那些 java.util.concurrent.atomic.* 类便是使用了这些封装后的接口，这些接口的提供者就是 Unsafe。

CAS 操作需要输入两个数值，一个旧值（期望操作前的值）和一个新值，在操作期间先比较下在旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换。CAS 操作是原子性的，所以多线程并发使用 CAS 更新数据时，可以不使用锁。JDK 中大量使用了 CAS 来更新数据而防止加锁（synchronized 重量级锁）来保持原子更新。

# Unsafe 类

Unsafe 是位于 sun.misc 包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升 Java 运行效率、增强 Java 语言底层资源操作能力方面起到了很大的作用。但由于 Unsafe 类使Java语言拥有了类似C语言指针一样操作内存空间的能力，这无疑也增加了程序发生相关指针问题的风险。在程序中过度、不正确使用 Unsafe 类会使得程序出错的概率变大，使得 Java 这种安全的语言变得不再“安全”，因此对 Unsafe 的使用一定要慎重。

Unsafe 类的设计者并不希望 Unsafe 被轻易的使用，尽管 Unsafe 里面的方法都是 public 的，但是并没有办法使用它们，JDK API 文档也没有提供任何关于这个类的方法的解释。总而言之，对于 Unsafe 类的使用都是受限制的，只有授信的代码才能获得该类的实例，当然 JDK 库里面的类是可以随意使用的。

Unsafe 类是单例实现的，提供静态方法 getUnsafe 获取 Unsafe 实例，当且仅当调用 getUnsafe 方法的类为引导类加载器所加载时才合法，否则抛出 SecurityException 异常。

```java
public final class Unsafe {
    // 单例对象
    private static final Unsafe theUnsafe;

    private Unsafe() { }

    @CallerSensitive
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();
        // 仅在引导类加载器`BootstrapClassLoader`加载时才合法
        if(!VM.isSystemDomainLoader(var0.getClassLoader())) {    
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }
}
```

想要获取 Unsafe 类实例有以下两种实现方案：

通过反射：

```java
private static Unsafe reflectGetUnsafe() {
    try {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        return (Unsafe) field.get(null);
    } catch (NoSuchFieldException | IllegalAccessException e) {
        log.error(e.getMessage(), e);
        return null;
    }
}
```

从getUnsafe方法的使用限制条件出发，通过Java命令行命令-Xbootclasspath/a把调用Unsafe相关方法的类A所在jar包路径追加到默认的bootstrap路径中，使得A被引导类加载器加载，从而通过Unsafe.getUnsafe方法安全的获取Unsafe实例：

```
java -Xbootclasspath/a: ${path} // path 为调用 Unsafe 相关方法的类所在 jar 包路径 
```

# Unsafe 对 CAS 的实现

Unsafe 类中只有三个跟 CAS 有关的方法，如下：

```java
/**
 * CAS 实现
 * 
 * @param o             包含要修改字段的对象
 * @param offset        要修改字段在该对象中的内存偏移量
 * @param expected      期望值
 * @param update        更新值
 * @return              更新成功 ? true : false
 */

public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object update);
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int update);
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);
```

写一个小例子演示怎么使用 Unsafe 的 CAS 操作：

```java
public static void main(String[] args) throws NoSuchFieldException {
    Unsafe unsafe = reflectGetUnsafe();

    User u = new User();
    u.setId(1001L);
    u.setName("zhangsan");
    u.setAge(18);

    Field id = User.class.getDeclaredField("id");
    Field name = User.class.getDeclaredField("name");
    Field age = User.class.getDeclaredField("age");

    long idOffset = unsafe.objectFieldOffset(id);
    long nameOffset = unsafe.objectFieldOffset(name);
    long ageOffset = unsafe.objectFieldOffset(age);

    unsafe.compareAndSwapLong(u, idOffset, 1001L, 1002L);
    unsafe.compareAndSwapObject(u, nameOffset, "zhangsan", "lisi");
    unsafe.compareAndSwapInt(u, ageOffset, 18, 20);

    System.out.println(u);
}

// 执行结果：
// User{id=1002, name='lisi', age=20}
```

{% note default %}
**CAS 存在的问题：**
{% endnote %}

1. **ABA 问题**
   因为 CAS 需要在操作值的时候，检查值有没有发生变化，比如没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用 CAS 进行检查时则会发现它的值没有发生变化，但是实际上却变化了。

   ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么 A->B->A 就会变成 1A->2B->3A。

2. **自旋开销过大**
    自旋CAS如果长时间不成功，会给 CPU 带来非常大的执行开销。

3. **只能够保证一个变量的原子操作**
   当对一个共享变量执行操作时，我们可以使用循环 CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候就可以用锁。

   从 Java 1.5 开始，JDK 提供了 AtomicReference 类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行 CAS 操作。

# 原子类对 CAS 的包装

原子类是 JUC 为我们提供的方便程序员使用 CAS 的工具类，位于 java.util.concurrent.atomic 包，该包下的所有类都是原子类。

接下来将把该包下的原子类分为：

- 基本数据类型原子类，以 `AtomicInteger` 为代表
- 数组类型原子类，以 `AtomicIntegerArray` 为代表
- 引用类型原子类，以 `AtomicReference` 为代表
- 原子更新字段类，以 `AtomicIntegerFieldUpdater` 为代表

以及解决了 ABA 问题的原子类：`AtomicStampedReference`，还有 JDK8 新增的 Adder 和 Accumulator

## AtomicInteger

AtomicInteger 的构造过程如下：

```java
// Unsafe 类实例，Unsafe CAS 所需要的偏移量
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

// 静态加载 value 字段相对当前对象的“起始地址”的偏移量
static {
    try {
        valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

// 被包装的 int 值
private volatile int value;

// 构造方法
public AtomicInteger() { }
public AtomicInteger(int initialValue) { value = initialValue; }
```

**常用 API：**

```java
public final int get() // 获取当前的值
public final boolean compareAndSet(int expect, int update) // CAS 更新，expect：预期值，update：更新值
public final int getAndSet(int newValue) // 获取当前的值，并设置新的值
public final int getAndIncrement() // 获取当前的值，并自增
public final int getAndDecrement() // 获取当前的值，并自减
public final int getAndAdd(int delta) // 获取当前的值，并加上预期的值
void lazySet(int newValue) // 最终会设置成 newValue,使用 lazySet 设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

常用 API 具体的使用方法这里就不展开了，JDK API 文档里面都有，其实现原理也比较好理解。

## AtomicIntegerArray

> 数组中的每个元素都具备原子性。

**常用方法：**

```java
public final boolean compareAndSet(int i, int expect, int update) // CAS 更新，i：数组下标
public final int get(int i) // 获取下标i的元素
```

**常用方法演示：**

```java
public static void main(String[] args) {
    AtomicIntegerArray array = new AtomicIntegerArray(new int[]{1, 2});
    array.compareAndSet(0, 1, 2);
    System.out.println(array.get(0));
}

// 执行结果：
// 2
```

## AtomicReference

> 使对象具备原子性的修改

**常用方法：**compareAndSet(V expect, V update)，与前文相同。

**常用方法演示：**

```java
public static void main(String[] args) {
    User u1 = new User("zhangsan");
    User u2 = new User("lisi");

    AtomicReference<User> atomicUser = new AtomicReference<>(u1);
    System.out.println(atomicUser.get());
    atomicUser.compareAndSet(u1, u2);
    System.out.println(atomicUser.get());
}

// 执行结果：
// User{name='zhangsan'}
// User{name='lisi'}
```

**两个注意点：**

1. AtomicArray 能够使数组中的每个元素都具有原子性，AtomicReference 可不是让对象里面的每个属性都具有原子性，仅仅是这个对象在修改的时候具有原子性。
2. 对于 AtomicReference 的使用，可能这里会有一个误区，认为 AtomicReference 对于一个对象的包装好像没什么用处。看完上面的例子之后可能认为不使用 AtomicReference 而直接让 `u1 = u2` 好像也能达到同样的效果，针对上面的这个例子这确实没错。**但是 `u1 = u2` 这不是一个原子操作！**具体原因在[Java 内存模型（JMM）](https://www.wrp.cool/posts/41498/)的 *「JMM 怎么解决原子性问题」* 小节中有具体介绍。

在开篇的时候提到了 CAS 常常会采用自旋来做为失败补偿机制，在这里演示一个 CAS 自旋锁：

```java
public class CASSpinLock {

    private AtomicReference<Thread> casSpinLock = new AtomicReference<>();

    public void lock() {
        Thread current = Thread.currentThread();
        while (!casSpinLock.compareAndSet(null, current)) { }
    }

    public void unlock() {
        Thread current = Thread.currentThread();
        sign.compareAndSet(current, null);
    }
}
```

## AtomicIntegerFieldUpdater

> 将一个对象中的某个属性升级为具有原子能力的属性

**使用场景：**

1. 在特定场景下才需要某个字段具有原子能力，如果一开始将该对象设计为原子对象，会给内存一定的压力。
2. 为做不到线程安全的类设置原子能力。

原子更新字段类的使用与前面的其他原子类有一些不同，主要就是在创建实例对象的时候不同，使用方式如下：

```java
public static void main(String[] args) {
    AtomicIntegerFieldUpdater<Data> atomicDataI = AtomicIntegerFieldUpdater.newUpdater(Data.class, "i");

    final Data data = new Data(0);
    atomicDataI.compareAndSet(data, 0, 1);
    System.out.println(atomicDataI.get(data));
}

private static class Data {
    volatile int i;

    public Data(int i) {
        this.i = i;
    }
}
```

**注意点：**

1. 原子更新字段类是将该字段升级为原子字段，这并不是针对某个对象的升级，而是针对于类的升级，被升级类的全部实例对象都可以使用该原子字段。
2. 创建后与一般的原子类使用无异。

## AtomicStampedReference

> 需要手动为每次 CAS 更新操作维护一个版本号，以此来解决 ABA 问题。

```java
public static void main(String[] args) throws InterruptedException {
    User u1 = new User("zhangsan");
    User u2 = new User("lisi");

    AtomicStampedReference<User> atomicUser = new AtomicStampedReference<>(u1, 1);
    Thread t = new Thread(() -> {
        atomicUser.compareAndSet(u1, u2, 1, 2);
        atomicUser.compareAndSet(u2, u1, 2, 3);
    });
    t.start();
    t.join();

    System.out.println(atomicUser.compareAndSet(u1, u2, 1, 2));
    System.out.println(atomicUser.compareAndSet(u1, u2, 3, 4));
}

// 运行结果：
// false
// true
```

> `AtomicStampedFieldUpdater` 网上看到很多文章都提到这个类说是啥带版本号原子更新字段啥的，但是在 oracle 文档却没有找到该类，可能是灵异事件吧。

## Adder 和 Accumulator

`LongAdder`、`LongAccumulator`、`DoubleAdder`、`DoubleAccumulator`，这四个类是在 JDK8 中新增的，接下来将主要分析 LongAdder 这个类，其他的几个类都差不多。引入 LongAdder 主要是为了优化 AtomicLong 在多线程并发情况下的效率，其背后的原理也是空间换时间。

AtomicLong 相对于 LongAdder 的缺点：速度慢，在多线程的情况下竞争同一个变量 `value` 导致出现大量线程自旋的情况。

```java
public static void main(String[] args) {
    LongAdder counter = new LongAdder();
    ExecutorService service = Executors.newFixedThreadPool(20);
    long start = System.currentTimeMillis();
    for (int i = 0; i < 10000; i++) {
        service.submit(new Task(counter));
    }
    service.shutdown();
    while (!service.isTerminated()) { }
    long end = System.currentTimeMillis();

    System.out.println(counter.sum());
    System.out.println("LongAdder耗时：" + (end - start));
}

private static class Task implements Runnable {

    private final LongAdder counter;

    public Task(LongAdder counter) {
        this.counter = counter;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            counter.increment();
        }
    }
}
```

在我的电脑上这段代码耗时大概是在 100ms 左右。同样的代码如果使用 AtomicLong 实现大概需要 1000ms 左右。二者在多线程的情况下差距十分明显。

LongAdder 的原理是：在最初无竞争时，只更新 base 的值，当有多线程竞争时通过分段的思想，让不同的线程更新不同的段，最后把这些段相加就得到了完整的 LongAdder 存储的值。

{% note modern default no-icon %}
**LongAdder 重要方法：**
{% endnote %}

**add(long x) 方法：**使用它可以使LongAdder中存储的值增加x，x可为正可为负。

```java
public void add(long x) {
    // as是Striped64中的cells属性
    // b是Striped64中的base属性
    // v是当前线程hash到的Cell中存储的值
    // m是cells的长度减1，hash时作为掩码使用
    // a是当前线程hash到的Cell
    Cell[] as; long b, v; int m; Cell a;
    // 条件1：cells不为空，说明出现过竞争，cells已经创建
    // 条件2：cas操作base失败，说明其它线程先一步修改了base，正在出现竞争
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        // true表示当前竞争还不激烈
        // false表示竞争激烈，多个线程hash到同一个Cell，可能要扩容
        boolean uncontended = true;
        // 条件1：cells为空，说明正在出现竞争，上面是从条件2过来的
        // 条件2：应该不会出现
        // 条件3：当前线程所在的Cell为空，说明当前线程还没有更新过Cell，应初始化一个Cell
        // 条件4：更新当前线程所在的Cell失败，说明现在竞争很激烈，多个线程hash到了同一个Cell，应扩容
        if (as == null || (m = as.length - 1) < 0 ||
            // getProbe()方法返回的是线程中的threadLocalRandomProbe字段
            // 它是通过随机数生成的一个值，对于一个确定的线程这个值是固定的
            // 除非刻意修改它
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            // 调用Striped64中的方法处理
            longAccumulate(x, null, uncontended);
    }
}
```

1. 最初无竞争时只更新base
2. 直到更新base失败时，创建cells数组
3. 当多个线程竞争同一个Cell比较激烈时，可能要扩容

> 请注意，这里的 casBase() 方法失败后，采用的是数组缓解多线程竞争的策略，再一次印证了自旋跟 CAS 没啥必须的关系。

> 具体的 longAccumulate 方法分析可以转至 [死磕 java并发包之LongAdder源码分析](https://zhuanlan.zhihu.com/p/65520633) 进行具体了解。大佬写的很好，是我道行不够，看不明白。

**sum()方法：**获取LongAdder中真正存储的值的大小，通过把base和所有段相加得到。

```java
public long sum() {
    Cell[] as = cells; Cell a;
    // sum初始等于base
    long sum = base;
    // 如果cells不为空
    if (as != null) {
        // 遍历所有的Cell
        for (int i = 0; i < as.length; ++i) {
            // 如果所在的Cell不为空，就把它的value累加到sum中
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    // 返回sum
    return sum;
}
```

可以看到 sum() 方法是把base和所有段的值相加得到，那么，这里有一个问题，如果前面已经累加到 sum 上的 Cell 的 value 有修改，不是就没法计算到了么？

答案确实如此，所以 LongAdder 可以说不是强一致性的，它是最终一致性的。

---

> **巨人的肩膀：**
> 
> - [Java魔法类：Unsafe应用解析](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)
> - [JUC原子类：CAS, Unsafe和原子类详解](https://pdai.tech/md/java/thread/java-thread-x-juc-AtomicInteger.html)
> - [死磕 java并发包之LongAdder源码分析](https://zhuanlan.zhihu.com/p/65520633)