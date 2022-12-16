---
title: Java：Lambda 表达式
abbrlink: 47234
date: 2021-05-23 09:12:50
categories:
 - Java
 - 高级特性
---

jdk1.8 之后才出现了 Lambda 表达式这个东西，这说明了 Lambda 表达式实际上并不是编程中必须掌握的一项技能，但是既然 jdk1.8 之后支持了lambda表达式，那么说明 Lambda 表达式肯定是会有其独特的用处。其实 Lambda 表达式最重要的就是让我们写的代码更加的优雅，看起来更加的舒服。同时，使用 Lambda 表达式也能在一定程度上少写一些代码，提高一些编程的效率。不过我还是认为 Lambda 表达式最重要的是让代码变得更加优雅。

关于 Lambda 表达式的一些基本的使用在网上实际上已经有了很多的博客或者教程，本文就不再赘述这些别人已经写过的东西了，主要还是想聊一下自己在学习 Lambda 表达式过程中的一些理解。

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    list.add(2021);
    list.add(0);
    list.add(5);
    list.add(23);
    list.forEach(item -> System.out.print(item));
}

// 输出：20210523
```
`item -> System.out.print(item)` 这就是一个最基本的 Lambda 表达式，这实际上是一个抽象方法的实现，只不过是写成这个样子，看起来更加优雅了。

```java
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```
上面的是 forEach() 方法的接口定义，具体的实现内容先不看，先看看 forEach 的参数列表中的 `Consumer<? super T> action` 这个东西，它叫函数式接口。

> 在《Java核心技术 卷一》这本书中对函数式接口的定义是这样的：
> 对于只有一个抽象方法的接口，需要这种接口的对象时，就可以提供一个 lambda 表达式。
> 这种接口称为**函数式接口**（functional interface）
> 
> 粗浅的理解可以是：一个接口中如果只定义了一个抽象方法的话，那么这个接口就是一个函数式接口。

看看这个 Consumer 所谓的函数式接口中唯一定义的一个抽象方法长什么样：

```java
void accept(T t);
```

上面说的 `item -> System.out.print(item)` 这个东西是对一个抽象方法的实现，实际上就是对 `accept` 这个抽象方法的实现，我以我的理解来尝试复原这个实现，将它变成我们平时见到的普通的方法实现。

```java
@Override
public void accept(T item) {
    System.out.print(item);
}
```

看起来就清晰多了，forEach() 方法将这个 Lambda 表达式（`item -> System.out.print(item)`）解析成了它所需要的参数（`Consumer<? super T> action`），也就是 Consumer 接口的实现，之后再调用这个方法来遍历List集合。

*当然具体的遍历方式我们也看到了，底层还是使用的forEach循环来遍历这个集合，所以在这里顺带提一下，除非遍历集合的内容只要一行代码就可以完成，像是我上面这样子，否则使用lambda表达式来遍历集合的话就不是很必要了。因为这并不会让我们的代码变得优雅或者是提高效率，反而平白的增加的可读成本，得不偿失。*

> 思考一个问题：
<font color = blue>如果有一个方法的代码刚刚好可以实现accept这个抽象接口，能不能直接把这个方法的代码作为accept的实现传递给forEach方法呢？</font>

还是上面的这个例子：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    list.add(2021);
    list.add(0);
    list.add(5);
    list.add(23);
    list.forEach(System.out::print);
}

// 输出：20210523
```

`System.out::print` 也是一种lambda表达式，在这段代码中，这两种实现方式的效果是完全一样的。

> `::`表示的是方法的引用，实际上就是将`System.out`对象中的`print`方法直接作为抽象方法`accept`的实现传到forEach中去。

结合这两个例子来看，Lambda 表达式中我们实际上只是传递的一个代码段，而不是接口的实现，实际会自动的根据上下文帮助我们封装成接口的实现以供方法调用，仅此而已。这就是 Lambda 表达式的真面目。

---

除了传递方法的引用进去，lambda表达式还可以直接传递一个构造器。像这样：

```java
/* 声明两个函数式接口 (此处借鉴 java.util.function.Supplier 函数接口)*/
@FunctionalInterface
interface Supplier1<T> {
    T get();
}

@FunctionalInterface
interface Supplier2<T> {
    T get(String str);
}

public class Main {

    // 两个构造方法
    public Main() { System.out.println("执行无参构造器"); }
    public Main(String str) { System.out.println("执行有参构造器"); }

    public static void main(String[] args) {
        Supplier1<Main> s1 = Main::new;
        Supplier2<Main> s2 = Main::new;

        s1.get();
        s2.get("");
    }
}

// 输出：
// 执行无参构造器
// 执行有参构造器
```

上面的代码就是调用了 Main 类中对应的构造器来构建一个 Main 的实例并返回，写法也非常的简单，直接双冒号调用new关键字即可。具体的调用哪一个构造器会根据上下文自动选择跟函数式接口参数对应的上的构造器。

上面的使用 Lambda 表达式创建的 Supplier1 对象和 Supplier2 对象的方式等价于下面这种方式：

```java
Supplier1<Main> s1 = new Supplier1<Main>() {
    @Override
    public Main get() {
        return new Main();
    }
};

Supplier2<Main> s2 = new Supplier2<Main>() {
    @Override
    public Main get(String str) {
        return new Main(str);
    }
};
```

---

**Lambda 表达式中的 this 指向：**

```java
public class Main {
    public Runnable test1() {
        return new Runnable() {
            @Override
            public void run() { System.out.println(this); }
            @Override
            public String toString() { return "Test"; }
        };
    }

    public Runnable test2() {
        return () -> { System.out.println(this); };
    }

    public static void main(String[] args) {
        Main main = new Main();
        new Thread(main.test1()).start();
        new Thread(main.test2()).start();
    }

    @Override
    public String toString() { return "Main"; }
}

// 输出：
// Test
// Main
```

借书里的一句话来说明一下 this 的指向：在 Lambda 表达式中，this 的使用并没有任何特殊之处。Lambda 表达式的作用域嵌套在 test2 方法中，与出现在这个方法中的其他位置一样，Lambda  表达式中 this 的语义并没有发生变化。

对于这个实际上很好理解，<font color=blue> Lambda 表达式中的 this 出现在任何地方都跟哪个地方本来的 this 是一样的，并没有因为 Lambda 表达式而发生不同。</font>这实际上也从另外的角度说明了 Lambda 表达式仅仅只是传递了一段代码过去，而没有做其他处理。

---

关于 Lambda 表达式有一个很经典的应用，跟着 Java8 一起来的还有 Stream 流操作，这是一种对于数组、集合等更方便更优雅的操作，Lambda 表达式跟 Stream 流操作配合起来能写出非常棒的代码。关于 Stream 流操作掘金上有一篇文章写的很好，[吃透JAVA的Stream流操作，多年实践总结](https://juejin.cn/post/7118991438448164878)。推荐大家都可以读一读。