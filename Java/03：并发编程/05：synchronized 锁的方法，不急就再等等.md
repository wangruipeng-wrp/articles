---
title: synchronized 锁的方法，不急就再等等
abbrlink: 11151
date: 2022-11-01 21:00:34
tags:
---

在 synchronized 的使用中，如果继续往下执行代码的条件不被满足的话可以先释放当前持有的锁对象再等等，等到执行条件被满足后再接着往下执行。

> 就像平时去医院排队看病一样，轮到我们了在医生问诊的过程中，可能会先让你先去做某一项检查，等检查完拿到检查结果之后再重新排队等待医生叫号。
> 排队看病的例子中，当我们在被医生问诊的时候，我们就独占了医生这把锁；需要做某一项检查的时候就是往下继续问诊的流程继续不了了；拿到检查结果后我们也要重新排队等待医生继续问诊。

在程序的世界里，我们使用 synchronized 锁来保护医生对象，一个医生同一时刻只能给一个病人问诊；调用 wait() 方法先去做检查；当问诊结束了调用 notify() 方法去通知下一位病人问诊。

# wait() 方法介绍

**wait() 方法的使用：**

```java
synchronized (object) {
    object.wait();
}

synchronized (Object.class) {
    Object.class.wait();
}

public synchronized void sync() throws InterruptedException {
    this.wait();
}

public static synchronized void sync() throws InterruptedException {
    Object.class.wait();
}
```

**wait() 方法的重载：**

- wait(long timeout)：在 timeout 毫秒内如果没有被 notify 唤醒则自行唤醒。
- wait(long timeout, int nanos)：在 1,000,000*timeout+nanos 纳米内如果没有被 notify 唤醒则自行唤醒。

# notify() 方法介绍

调用 wait() 方法当前线程会放弃 synchronized 锁，相对应的调用 notify() 方法之后原来放弃 synchronized 锁的线程就会重新去竞争 synchronized 锁，竞争到锁的线程可以继续执行。

notify() 方法还有一个兄弟方法叫做 notifyAll()，notify() 方法只会唤醒被 wait() 阻塞的线程之一；而 notifyAll() 会唤醒全部被 wait() 阻塞的线程。

被 wait() 方法阻塞的线程想要被唤醒必须是另外的一个线程去调用了 notiify() 并且轮到本线程被唤醒，或者是直接调用 notifyAll()。

**代码示例：**

```java
public class Demo implements Runnable {

    private final int num;
    public Demo(int num) { this.num = num; }

    private static final Object LOCK = new Object();
    private static final List<Integer> waitList = new ArrayList<>(50);
    private static final List<Integer> notifyList = new ArrayList<>(50);

    @Override
    public void run() {
        synchronized (LOCK) {
            try {
                waitList.add(num);
                LOCK.wait();
                notifyList.add(num);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 50; i++) {
            new Thread(new Demo(i)).start();
            Thread.sleep(10); // 一个个去睡觉
        }
        // notifyDemo();
        // notifyAllDemo();
        System.out.println(waitList);
        System.out.println(notifyList);
    }

    public static void notifyDemo() throws InterruptedException {
        for (int i = 0; i < 50; i++) {
            synchronized (LOCK) {
                LOCK.notify();
            }
            Thread.sleep(10); // 一个个来唤醒
        }
    }

    public static void notifyAllDemo() throws InterruptedException {
        synchronized (LOCK) {
            LOCK.notifyAll();
        }
        Thread.sleep(50); // 等待全部唤醒
    }
}
```

在 main() 方法中调用 notify() 方法的执行结果是：

```
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

在 main() 方法中调用 notifyAll() 方法的执行结果是：

```
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
[9, 8, 7, 6, 5, 4, 3, 2, 1, 0]
```

很明显，notify() 是顺序唤醒线程的，而 notifyAll() 是倒序唤醒线程的。如果想要深究这部分原因，进入到 Object 类中看到 wait()、notify()、notifyAll() 这三个方法都是被 native 修饰的方法。再深入就是 jvm 的 C++ 实现了，包括还得看 synchronized 的实现等等。这部分内容暂时道行还不够，就先不做介绍了，暂时就先记住这么个现象吧。

如果业务上需要对线程的唤醒顺序有要求的话，可以分用多个锁来指定唤醒对应线程。

```java
public static void main(String[] args) {
    final Object LOCK = new Object();

    new Thread(() -> {
        synchronized (LOCK) {
            try {
                LOCK.wait();
                System.out.println("被唤醒。");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }).start();

    new Thread(() -> {
        synchronized (LOCK) {
            LOCK.notify();
            StringBuilder str = new StringBuilder();
            str.append("[");
            for (int i = 0; i < 10; i++) {
                str.append(i + 1);
                if (i != 9)
                    str.append(",");
            }
            str.append("]");
            System.out.println(str);
        }
    }).start();
}
```

再看上面这段代码，输出结果是：
```java
[1,2,3,4,5,6,7,8,9,10]
被唤醒。
```

显然对于 notify() 和 wait() 之间的执行顺序，调用 notify() 或者 notifyAll() 之后并不会马上释放本线程的 monitor 锁，而是等待被线程执行完毕之后再释放 monitor 锁给被 wait() 阻塞的线程去竞争。

# 问诊案例

```java
public class SeeDoctor {

    final Doctor LOCK = new Doctor();

    public static void main(String[] args) {
        synchronized(LOCK) {
            System.out.println("Hello doctor.");
            while (needDoCheck()) {
                LOCK.notifyAll();
                LOCK.wait();
            }
            System.out.println("You're healthy");
            LOCK.notifyAll();
        }
    }

    public static boolean needDoCheck() {
        if (checkIsDone()) {
            return false;
        }
        if (isChecking()) {
            return true;
        }
        new Thread(() -> {
            System.out.println("is checking...");
            System.out.println("wait a minute");
        }).start();
        return false;
    }
}
```

以上是问诊案例的大概流程，有三个地方需要注意：

1. 判断是否需要等待应该使用 while 循环判断，因为很可能被唤醒的线程检查结果还没出来，还是需要继续等待。

3. 尽量使用 notifyAll() 方法唤醒，因为不能确定被唤醒线程能否往下继续执行。如果被唤醒线程因条件不满足继续等待，那还有其他线程可以继续抢锁，而 notify() 在这种情况下会导致所有线程都陷入等待无法唤醒。