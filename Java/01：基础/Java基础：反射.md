---
title: Java：反射
abbrlink: 25346
date: 2020-10-02 19:45:30
categories:
 - Java
 - 高级特性
---

# 概述
首先来看一下百度百科中对于Java反射的定义，[JAVA反射机制，百度百科](https://baike.baidu.com/item/JAVA%E5%8F%8D%E5%B0%84%E6%9C%BA%E5%88%B6/6015990?fr=aladdin)

> Java的反射（reflection）机制是指在程序的运行状态中，可以构造任意一个类的对象，可以了解任意一个对象所属的类，可以了解任意一个类的成员变量和方法，可以调用任意一个对象的属性和方法。这种动态获取程序信息以及动态调用对象的功能称为Java语言的反射机制。反射被视为动态语言的关键。
<!-- more -->
> Java反射机制主要提供了以下功能： 在运行时判断任意一个对象所属的类；在运行时构造任意一个类的对象；在运行时判断任意一个类所具有的成员变量和方法；在运行时调用任意一个对象的方法；生成动态代理。

> 有时候我们说某个语言具有很强的动态性，有时候我们会区分动态和静态的不同技术与作法。我们朗朗上口动态绑定（dynamic binding）、动态链接（dynamic linking）、动态加载（dynamic loading）等。然而“动态”一词其实没有绝对而普遍适用的严格定义，有时候甚至像面向对象当初被导入编程领域一样，一人一把号，各吹各的调。

> 一般而言，开发者社群说到动态语言，大致认同的一个定义是：“程序运行时，允许改变程序结构或变量类型，这种语言称为动态语言”。从这个观点看，Perl，Python，Ruby是动态语言，C++，Java，C#不是动态语言。

> 尽管在这样的定义与分类下Java不是动态语言，它却有着一个非常突出的动态相关机制：Reflection。这个字的意思是“反射、映象、倒影”，用在Java身上指的是我们可以于运行时加载、探知、使用编译期间完全未知的classes。换句话说，Java程序可以加载一个运行时才得知名称的class，获悉其完整构造（但不包括methods定义），并生成其对象实体、或对其fields设值、或唤起其methods。这种“看透class”的能力（the ability of the program to examine itself）被称为introspection（内省、内观、反省）。Reflection和introspection是常被并提的两个术语。

上面这几段话出自百度百科，由此可以看出，Java反射机制的重要性。由于反射机制的存在使得Java语言变身成为一门准动态语言。很多主流框架中也是大量的使用了反射技术，像是我们说的Spring就是基于反射 + 配置 + 工厂的形式实现的。

# Class类
想要了解反射机制，首先来了解一个类`java.lang.Class`反射机制中最重要的一个类，Class类（描述类的类）。

**Java中的类：**可以继承某个类，可以实现某一些接口，可以在类中定义类的属性，方法，和构造器。

由此，Class类（描述类的类）主要就是描述了类继承了哪个父类，实现了那些接口，定义了那些方法和属性，以及类中的构造器。

**小结：**类的声明、类中的属性和方法、类的构造器是 `Class` 类主要的描述对象。

---

既然Class类是描述类的类，那么就表明了Class类也是一个Java类，也必须遵守Java中关于类的规范，这一点不会变。

来分析一下Class类，首先看一下构造函数。
![20200921203210]( https://wrp-blog-image.oss-cn-beijing.aliyuncs.com/blog-images/20200921203210.png)
这是jdk1.8中Class类的构造函数，这是一个私有化的构造函数，意味着在外界是不能直接创建Class类的实例的。

实际上，Class类的实例也并不是创建出来的，Class类的实例是JVM在加载每个class字节码文件时自动生成的实例。

每个Class类对应的都是被JVM所加载的一个个class文件，相同的，每个class文件也都有一个唯一与之对应的Class实例。

# 如何获取Class类的实例
这里将通过一个User类来演示四种获取Class实例的方式。
```java
package cool.wrp.reflex;

public class User {
}

class Test{
    public static void main(String[] args) throws ClassNotFoundException {
        // 1.类名.class
        Class<User> userClass1 = User.class;

        // 2.实例.getClass()
        User user = new User();
        Class userClass2 = user.getClass();

        // 3.Class.forName("全限定类名")
        Class userClass3 = Class.forName("cool.wrp.reflex.User");

        // 4.类加载器
        ClassLoader classLoader = user.getClass().getClassLoader();
        Class userClass4  = classLoader.loadClass("cool.wrp.reflex.User");

        System.out.println(userClass1.hashCode());
        System.out.println(userClass2.hashCode());
        System.out.println(userClass3.hashCode());
        System.out.println(userClass4.hashCode());
    }
}
```
运行main方法可以发现，四个Class实例的hashCode都是相同的，这证明了每个class字节码文件都有且仅有唯一一个与之对应的Class实例。

以上的四种方式，我们在实际开发中应该尽可能的使用第一种方式去获得Class类的实例，因为这种方式的效率是最高的。

# Class类是怎么描述类的
<p style="text-indent:2em">
在这里引用 <b>Java核心技术卷一 5.7.4</b> 中的内容来讲述Class类是怎么描述类的。
</p>

<p style="text-indent:2em">
在 java.lang.reflect 包中有三个类 Firld、Method 和 Construct 分别用于描述类的字段、方法和构造器。这三个类都有一个叫做 getName 的方法，用来返回字段、方法或构造器的名称。Field 类还有一个 getType 方法，用来描述字段类型的一个对象，这个对象的类型同样是Class。Method 和 Constructor 类有报告参数类型的方法，Method类还有一个报告返回类型的方法。这三个类都有一个名为 getModifiers 的方法，它将返回一个整数， 用不同 0/1 位描述所使用的修饰符，如 public 和 static。另外，还可以利用 java.lang.refkect 包中的 Modifier 类的静态方法分析 getModifiers 返回的这个整数，例如，可以使用 Modifier 类中的 isPublic、isPrivate 或 isFinal 判断方法或构造器是 public、private 还是 final。我们需要做的就是在 getModifiers 返回的整数上调用 Modifier 类中适当的方法，另外，还可以利用 Modifier.toString 方法将修饰符打印出来。
</p>

<p style="text-indent:2em">
Class 类中的 getFields、getMethods 和 getConstructors 方法将分别返回这个类支持的 <b>公共</b> 字段、方法和构造器的数组，其中包括超类的公共成员。Class 类的 getDeclareFields、getDeclareMethods 和 getDeclaredConstrors方法将分别返回类中声明的全部字段、方法和构造器的数组，其中包括私有成员、包成员和受保护成员，但不包括超类的成员。
</p>

---

**来看一个书中的例子加深对这段话的理解**

```java
public static void main(String[] args) throws ClassNotFoundException {

    // read class name from user input
    System.out.println("class name:");
    Scanner scanner = new Scanner(System.in);
    String className = scanner.next();

    // print class name and super class name (if != Object)
    Class cl = Class.forName(className);
    Class supercl = cl.getSuperclass();
    String modifiers = Modifier.toString(cl.getModifiers());
    if (modifiers.length() > 0)
        System.out.print(modifiers + " ");
    System.out.print("class " + className);
    if (supercl != null && supercl != Object.class)
        System.out.print(" extends " + supercl.getName());

    System.out.print("\n{\n");
    printConstructors(cl);
    System.out.println();
    printMethods(cl);
    System.out.println();
    printFields(cl);
    System.out.println("\n}\n");
}
```
简单分析一下以上的代码，首先是一个接收用户在控制台的输入，通过scanner对象读取用户从控制台输入的一个全限定类名。之后通过 Class.forName 的形式去获取到用户输入类名对应的Class类对象。获取到Class对象之后再去获取父类和修饰符。

如果需要判断类使用了什么修饰符可以使用 `Modifier.isPublic(cl.getModifiers())` 来判断类是否被 public 所修饰。以此类推，判断是否被 private 所修饰则使用 `isPrivate`，是否 final 则 `isFinal`等等。

另外可以使用 `getInterfaces()` 方法去获取到一个类所实现的接口有哪些。由于Java是支持多实现的，所以该方法实际上返回的是一个泛型为Class类型的数组，用来描述类所实现的接口。

> 以上是Class对象如何描述一个Java类声明的内容，接下来分析Class对象是如何去描述类的构造器、方法和属性的。

```java
/**
 * Prints all constructors of a class
 *
 * @param cl a class
 */
public static void printConstructors(Class cl) {
    Constructor[] constructors = cl.getDeclaredConstructors();

    for (Constructor c : constructors) {
        String name = c.getName();
        System.out.print("\t");
        String modifiers = Modifier.toString(c.getModifiers());
        if (modifiers.length() > 0) System.out.print(modifiers + " ");
        System.out.print(name + "(");

        // print parameter type
        Class[] paramTypes = c.getParameterTypes();
        for (int j = 0; j < paramTypes.length; j++) {
            if (j > 0) System.out.print(", ");
            System.out.print(paramTypes[j].getName());
        }
        System.out.println(");");
    }
}
```
```java
/**
 * Prints all method of a class
 *
 * @param cl a class
 */
public static void printMethods(Class cl) {
    Method[] methods = cl.getDeclaredMethods();

    for (Method m : methods) {
        Class retType = m.getReturnType();
        String name = m.getName();

        System.out.print("\t");
        // print modifiers, return type and method name
        String modifiers = Modifier.toString(m.getModifiers());
        if (modifiers.length() > 0) System.out.print(modifiers + " ");
        System.out.print(retType.getName() + " " + name + "(");

        // print parameter types
        Class[] paramTypes = m.getParameterTypes();
        for (int j = 0; j < paramTypes.length; j++) {
            if (j > 0) System.out.print(", ");
            System.out.print(paramTypes[j].getName());
        }
        System.out.println(");");
    }
}
```
```java
/**
 * Prints all fields of a class
 *
 * @param cl a class
 */
public static void printFields(Class cl) {
    Field[] fields = cl.getDeclaredFields();

    for (Field f : fields) {
        Class type = f.getType();
        String name = f.getName();
        System.out.print("\t");
        String modifiers = Modifier.toString(f.getModifiers());
        if (modifiers.length() > 0) System.out.print(modifiers + " ");
        System.out.println(type.getName() + " " + name + ";");
    }
}
```
有兴趣的同学可以执行一遍以上的代码，可以更清楚的展示Class对象是如何去描述一个类的。

---

书中的这个例子已经是很详细的讲明白了Class类是如何去描述一个类的，但这里还有一点补充，就是**Class类是如何去描述类中的注解的**。

我们使用 [【Java基础】](https://blog.wrp.cool/posts/28807/) 注解一文开头的注解例子做演示。
```java
@MyAnnotation(value = "MyAnnotation's value", name = "MyAnnotation's name")
class TargetClass { }

public class Test {
    public static void mainn(String[] args) {
        Class<?> cl = TargetClass.class;
        
        // 获取注解对象
        MyAnnotation myAnnotation = cl.getAnnotation(MyAnnotation.class);
        // 打印注解中的内容
        System.out.println(myAnnotation.value());
        System.out.println(myAnnotation.name());
    }
}
```
> 运行main方法：
`MyAnnotation's value`
`MyAnnotation's name`

> 同样的 `Constructor、Method、Field` 对象也有 `getAnnotation` 方法。

# Class类是怎么操作类的
操作类也是分别对应着Java中对类的操作，首先是类的实例化，然后可以**设置类的属性、获取类的属性、调用类的方法**。而属性和方法又分为**公共、私有和静态**

### 类的实例化
通过反射实例化的类也是需要调用构造器的，构造方法分为有参构造器和无参构造器。

使用反射实例化一个类有两种方式，一个是调用**Class对象的newInstance()方法**，另一个是调用**Constructor对象的newInstance()方法**。

**实验环境**
```java
class TargetClass {
    public TargetClass() throws NullPointerException {
        throw new NullPointerException();
    }

    public TargetClass(String str) {
        System.out.println(str);
    }
}
```

**Class 对象的 newInstance() 方法**
```java
public static void main(String[] args) {
    try {
        TargetClass.class.newInstance();
    } catch (InstantiationException | IllegalAccessException e) {
        e.printStackTrace();
    }
    
    System.out.println("main方法运行完毕");
}
```
`运行main方法抛出空指针异常`

**Constructor 对象的 newInstance() 方法**
```java
public static void main(String[] args) {
    Class<?> cl = TargetClass.class;

    // 无参构造器
    try {
        cl.getDeclaredConstructor().newInstance();
    } catch (InstantiationException | IllegalAccessException | InvocationTargetException | NoSuchMethodException e) {
        e.printStackTrace();
    }

    // 有参构造器
    try {
        cl.getDeclaredConstructor(String.class).newInstance("我是有参构造器实例化的对象");
    } catch (InstantiationException | IllegalAccessException | InvocationTargetException | NoSuchMethodException e) {
        e.printStackTrace();
    }

    System.out.println("main方法运行完毕");
}
```
> 运行main方法
`抛出由java.lang.NullPointerException引起的java.lang.reflect.InvocationTargetException`
打印：`我是有参构造器实例化的对象`
打印：`main方法运行完毕`

**小结：**
1. **Class 对象的 newInstance() 方法**无法调用有参构造函数，只能调用无参构造器
2. **Class 对象的 newInstance() 方法**无法捕获构造器中的异常，而**Constructor 对象的 newInstance() 方法**会将构造器中的异常封装成 `java.lang.reflect.InvocationTargetException` 异常
3. 建议使用**Constructor 对象的 newInstance() 方法**

---

> 在这里首先引入一个简单的概念，**显示参数**和**隐式参数**。在这里先简单的理解为调用属性的对象。

**显示参数：**在方法中明确定义的参数为显示参数

**隐式参数：**未在方法是定义的，但的确又动态影响到程序运行的“参数”

```java
public class Test {
    public static void main(String[] args) {
        User user = new User();

        user.setName("张三");
    }
}

class User {
    private String name;
    public void setName(String name) { this.name = name; }
}
```
例如以上代码 `user` 对象为隐式参数，实际传入的 `"张三"` 字符串为显示参数。

---

### 操作属性

**实验环境**
```java
// 目标类
class TargetClass {
    public String pubAttr;
    static String staAttr;
    private String priAttr;
}

public class Test {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        Class<?> cl = TargetClass.class;

        // 实例化一个对象作为隐式参数
        TargetClass tc = new TargetClass();

        // 获取属性
        Field pubAttr = cl.getDeclaredField("pubAttr");
        Field staAttr = cl.getDeclaredField("staAttr");
        Field priAttr = cl.getDeclaredField("priAttr");
    }
}
```

**公共属性**
> 实际上，Java中的非静态属性都是挂载在对应的对象上的。反射也没有例外，所以我们如果想要通过反射去操作一个属性，我们同样是需要给这个属性一个可以挂载的地方，也就是一个对象，这个对象也就是前文提到的隐式参数。
```java
// 设置公共属性的值
pubAttr.set(tc, "我是公共属性");
// 获取公共属性的值
System.out.println(pubAttr.get(tc));
```
`输出：我是公共属性`

上面的两行代码中的tc就是对应的对象挂载的地方。在这里我将其理解为pubAttr属性的隐式参数。

**静态属性**
> 刚刚说的Java中的非静态属性都是挂载在对象上的，而静态属性与Class对象一样是存储在方法区中的。所以不需要挂载的对象，在传参数的时候直接传一个null对象即可。
```java
// 设置静态属性的值
staAttr.set(null, "我是静态属性");
// 获取静态属性的值
System.out.println(staAttr.get(null));
```
`输出：我是静态属性`

**私有属性**
> Java中类的私有属性是无法通过外部去设置和获取的，而反射可以改变这一点，这同时也正是反射的强大之处。只需要一行代码设置访问权限即可访问类的私有属性。

```java
// 授权访问
priAttr.setAccessible(true);
// 设置私有属性的值
priAttr.set(tc, "我是私有属性");
// 获取私有属性的值
System.out.println(priAttr.get(tc));
```
`输出：我是私有属性`

### 操作方法

操作方法和操作属性实际上差不多，也都分共有、私有、静态方法。不同的是，在获取Method对象时需要指定参数列表，执行Method方法时也需要传入对应方法的参数。

```java
class TargetClass {
    public void pubMethod(String name) { System.out.println("我是" + name); }

    static void staMethod(String name) { System.out.println("我是" + name); }

    private void priMethod(String name) { System.out.println("我是" + name); }
}

public class Reflex {
    public static void main(String[] args) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
        Class<?> cl = TargetClass.class;

        TargetClass tc = (TargetClass) cl.newInstance();
 
        // 获取公共方法
        Method pubMethod = cl.getDeclaredMethod("pubMethod", String.class);
        // 执行公共方法
        pubMethod.invoke(tc, "公共方法");

        // 获取静态方法
        Method staMethod = cl.getDeclaredMethod("staMethod", String.class);
        // 执行静态方法
        staMethod.invoke(null, "静态方法");

        // 获取私有方法
        Method priMethod = cl.getDeclaredMethod("priMethod", String.class);
        // 私有方法授权
        priMethod.setAccessible(true);
        // 执行私有方法
        priMethod.invoke(tc, "私有方法");
    }
}
```