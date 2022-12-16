---
title: Java并发：ThreadLocal
abbrlink: 12951
date: 2022-07-16 14:58:24
description: ThreadLocal 是线程池的好朋友，它可以让每个线程都拥有自己的对象，而不是去跟别的线程共享同一个对象，以此达到线程安全。
categories:
  - Java
  - 并发
---

在线程池中必不可免的需要面对线程安全问题，在并发执行同一个任务的情况下，肯定需要访问到一些公共的对象，也就是会出现多个线程访问同一个对象的情况，这就涉及到了线程安全的问题。

这个时候就需要为线程池中的每一个线程都准备一个单独的变量供线程池中的每个线程去使用了。

{% note warning modern no-icon %}
于是：ThreadLocal 登场了。
{% endnote %}

说说我的理解吧，我认为 ThreadLocal 其实是一个封装的工具，可以使用 ThreadLocal 将线程池中各个线程都要访问到的对象封装起来，之后各个线程去访问被包装对象的时候，ThreadLocal 会生成一个这个对象的副本，也就是重新创建这个对象给线程去使用，以此将每个线程隔离开来，使得每个线程都能够拥有独属于自己的对象。

基于 ThreadLocal 为每个线程设置独享对象的特性，可以衍生出两个用法：

- 隔离共享对象，用于保证线程安全
- 设置线程内的全局变量，在同一个线程内免去了参数传递的麻烦

# ThreadLocal 用法一：隔离对象，保证线程安全

```java
import java.text.DateFormat;
import java.text.SimpleDateFormat;
 
public class DateUtils {
    public static final ThreadLocal<DateFormat> threadLocal = new ThreadLocal<DateFormat>(){
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };

    // 上面的代码的简写形式
    // public static final ThreadLocal<DateFormat> threadLocal = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

    public static void main(String[] args) {
        // 使用方式
        DateFormat dateFormat = DateUtils.threadLocal.get();
    }
}
```

由于 SimpleDateFormat 对象并不是线程安全的，如果在线程池中需要用到 SimpleDateFormat 对象的话，是不能让所有线程共用同一个对象的，那样会出现线程不安全问题。但如果线程每次运行都创建一个新的 SimpleDateFormat 对象的话，那么又太浪费资源了。

实际上这里还有一个解决方式：把用到 SimpleDateFormat 对象的地方都抽取出来加上 synchronized 关键字，把 SimpleDateFormat 对象给锁上，但是每个线程要使用的时候又都要去排队，这也并不是一个好方法，特别是线程池在执行的情况下，并发量很大，排队的时间就会很长。

> 使用 ThreadLocal 之后是一个什么样的流程呢？

1. 线程需要使用 SimpleDateFormat 对象
2. 调用 initialValue() 方法，生成一个 SimpleDateFormat 对象
3. 给到需要的线程去使用，之后再保存在线程中，下次再要用了直接拿出来就可以了
4. 也就是一个懒加载的方式

# ThreadLocal 用法二：设置线程内的全局变量

```java
private static final ThreadLocal<User> userContextHolder = new ThreadLocal<>();

public static void main(String[] args) {
    new Interceptor().checkingLogin(new User("张三"));
}

static class Interceptor {
    public void checkingLogin(User user) {
        userContextHolder.set(user);
        System.out.println(user);
        new Controller().getList();
    }
}

static class Controller {
    public void getList() {
        User user = userContextHolder.get();
        System.out.println(user);
        new Service().getList();
    }
}

static class Service {
    public void getList() {
        User user = userContextHolder.get();
        System.out.println(user);

        // 防止内存泄漏
        userContextHolder.remove();
    }
}

static class User {
    String name;
    public User(String name) { this.name = name; }

    @Override
    public String toString() { return "User{name='" + name + "'}"; }
}
```

可以利用 ThreadLocal 在线程中传递参数，免去了到处传参的麻烦，使用的话就像上面的这段代码一样就可以了。

{% note danger modern %}
**存在内存泄漏的风险，使用完一定要调用 remove() 释放 user 对象。**
{% endnote %}

# ThreadLocal 是怎么做到线程隔离的

![ThreadLocal原理图](https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/ThreadLocal原理图.png)

1. 每个 Thread 都有一个唯一对应的 ThreadLocalMap
   - ThreadLocalMap 以 ThreadLocal 对象为键，值是 initialValue() 或者 set() 方法中我们自己设置的
2. 首次调用 get() 方法时，ThreadLocal 会为当前线程初始化 ThreadLocalMap
3. 再调用 initialValue() 方法，返回对应的我们设置的对象，以该对象为值，ThreadLocal 对象为键存储至 ThreadLocalMap
4. 再返回该对象给线程使用

```java
public T get() {

    // 获取线程自己的 ThreadLocalMap
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);

    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }

    // 初始化 ThreadLocalMap
    return setInitialValue();
}

private T setInitialValue() {

    // 调用重载的 initialValue() 方法获取需要设置的值
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

---

{% note info modern %}
**小结：**
{% endnote %}
> 1. ThreadLocal 中的值是保存在 Thread.ThreadLocalMap 中的，这可能也是为什么叫 ThreadLocal 吧
> 2. 取值的方式是懒加载的，实际上也只能懒加载。
>    我们可能会定义很多个 ThreadLocal 对象出来用，但并不是每个线程都需要用到全部的 ThreadLocal，没到要用的时候程序也无法预先判断这个线程需要用到哪些 ThreadLocal。


# ThreadLocalMap

> ThreadLocalMap 是 ThreadLocal 的一个静态内部类，没有实现 Map 接口，底层由 Entry 数组实现。

- ThreadLocalMap 键：ThreadLocal 对象
- ThreadLocalMap 值：通过 initialValue() 或 set() 方法传入的对象

{% note info modern no-icon %}
**Entry 类：**
{% endnote %}

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

ThrealLocalMap 键值对实现的具体对象，需要注意的是 `k` 是一个弱引用。

> **Java 中的四种引用类型：**
> - **强引用：**我们常常 new 出来的对象就是强引用类型，只要强引用存在，垃圾回收器将永远不会回收被引用的对象，哪怕内存不足的时候
> - **软引用：**使用 SoftReference 修饰的对象被称为软引用，软引用指向的对象在内存要溢出的时候被回收
> - **弱引用：**使用 WeakReference 修饰的对象被称为弱引用，只要发生垃圾回收，若这个对象只被弱引用指向，那么就会被回收
> - **虚引用：**虚引用是最弱的引用，在 Java 中使用 PhantomReference 进行定义。虚引用中唯一的作用就是用队列接收对象即将死亡的通知

往 ThreadLocalMap 中添加元素：set() 方法

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;

    // 通过 hash 找到 key 在 Entry[] 的第一个下标
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    // 解决 hash 冲突
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get(); // get() 方法由 WeakReference 继承自 Reference 而来

        // 如果当前 ThreadLocal 已经存在，则覆盖
        if (k == key) {
            e.value = value;
            return;
        }

        // 如果找到的 e 不为空，但是 k 为空，则证明该 k 被 gc 回收
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 找到可插入位置，新建元素
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

> ThreadLocalMap 的冲突处理：往后继续找到合适的位置插入，而不是在该位置拉一个链表或者红黑树。

从 ThreadLocalMap 中取出元素：getEntry() 方法

```java
// 找到第一个 key 比较与传入的 key 是否相等，相等直接返回，不相等由哈希冲突策略继续往下找
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

先找到ThreadLocal的索引位置, 如果索引位置处的entry不为空并且键与threadLocal是同一个对象, 则直接返回; 否则去后面的索引位置继续查找。

---

# 总结

1. ThreadLocal 是 ThreadPool 的好朋友，借助 ThreadLocal 可以实现多线程环境下的线程安全，或者为单独的线程设置全局变量
2. ThreadLocal 是一个将线程不安全对象变成线程安全的工具，具体将被包装的对象为每个线程都做一份拷贝，大家各自用各自的变量，不争不抢，线程就安全了
3. Thread 会维护自己的 ThreadLocalMap，以 ThreadLocal 为键，set() 或 initialValue() 中的返回对象为值
4. ThreadLocalMap 是 ThreadLocal 的静态内部类，不实现 Map 接口
5. ThreadLocalMap 处理哈希冲突的策略是：向后移位，直到找到合适位置为止

---
> **巨人的肩膀：**
> [Java 并发 - ThreadLocal详解](https://pdai.tech/md/java/thread/java-thread-x-threadlocal.html)
> [万字解析 ThreadLocal 关键字](https://javaguide.cn/java/concurrent/threadlocal.html)