---
title: JavaScript：数据类型
abbrlink: 24923
date: 2020-05-22 21:30:45
categories:
 - JavaScript
---

我们所使用的编程语言，不管是js也好或者是其他的一些语言，例如：Java、C++。我们在编写程序的时候实际上都是在对数据进行操作。如果脱离了数据，那么我们编写的程序将变得毫无意义。甚至脱离了数据之后我们将无法编程。由此，数据类型便成了重中之重。

# 一、js数据类型简介
1. js的数据类型主要分为两大类，原始类型（即基本数据类型）和对象类型（即引用数据类型）。
2. js中的基本数据类型可以分为5种：Number、String、Boolean、Undefined、Null。
3. js中的引用数据类型也就是对象类型Object，主要是Object、Array、Function这几种。

# 二、基本数据类型和引用数据类型的区别
首先这里需要先简单的介绍一下堆内存和栈内存。计算机的内存中有堆内存和栈内存，栈内存呢是一个一个排列下去的是有序的，也是固定大小的，像是数组一样；而堆内存呢则刚好是与栈内存相反，堆内存中的数据是无序的，也是不固定大小的。
## 堆栈内存的优缺点
栈内存由于是有序的所以在查询数据的时候直接按照排序的序号去查的话是比较快的，但是由于固定大小的原因呢，栈内存无法随心所欲的存储数据。堆内存由于不固定大小，所以在其中存储的数据是比较方便的，也就是想怎么存就怎么存，但是因为堆内存中的数据是无序的，所以在查找的时候就是要比较慢一些。  
> 总结：就是栈内存优点是查询快，存储不方便；而堆内存则是存储方便，但查询慢。  

*这么说可能还是有点不太形象，举个例子吧。栈内存中的空间呢就像是现实生活中的商品房一样，一个一个的房子都是开发商建好的，每个房子的大小是固定的，每个房子也有对应的门牌号，比如某某单元的几零几这样，查找起来呢是比较方便。堆内存呢就像是一栋一栋的独栋别墅一样，没有一个集中的管理，但是大小不固定，可能是有一块地用来建别墅，我今天打算建个100平米的别墅，明天我就可能扩建到200平米了，甚至是再往上多建几层。*    

既然堆内存和栈内存有这样的优缺点，那么何不将两者的优点结合起来，在栈内存中存放指向堆内存中的指针。这样可以即查询快又存储数据方便。这样的数据也就是上面说的引用数据类型。我们待会细说。

# 三、基本数据类型的特点
1. 基本数据类型的赋值仅仅只是简单的值传递：
如果需要从一个变量向另一个变量去赋一个基本数据类型的值，那么会将其中基本变量的值生成一个副本再将这个副本的值直接传递过去，而不是引用同一个值。例如：
```JavaScript
var a = 18;
var b = a;
console.log(a); // 输出18
console.log(b); // 输出18
a++;
console.log(a); // 输出19
console.log(b); // 输出18
```
在上述的例子中，一开始先将变量`a`的值赋给了变量`b`，这时变量`a`的值和变量`b`的值都是18，然后我们再将变量`a`的值进行`++`操作，变量`a`的值变成了19，但是变量`b`的值任然还是18，由此可见变量`a`和变量`b`的值是完全互不相干的，没有关系的。

    *举个例子吧，就比如我们平时在使用`ctrl + c`和`ctrl + v`复制黏贴的时候是一样的，我们复制的文本并不会随着原来文本的改变而改变，同样原来的文本也不会因为被复制的文本的改变而发生任何变化。同理，变量之间的赋值也是一次简单的“`ctrl + c`和`ctrl + v`”而已。*
2. 基本数据类型的值是固定的，无法修改的。比如：
```JavaScript
var str = 'string';
var anotherStr = str.replace('s','');
console.log(str); // 输出 string
console.log(anotherStr); // 输出 tring
```
在上面的这个例子中我们可以看到，字符串`str`的值一直都是`string`，尽管我们调用的`replace`方法将其中的`s`替换成了空字符串，但是实际上的`str`的值并没有被修改，任然是`string`。修改后的值传回来了一个新的变量，这个变量的值才是`tring`，然而这是一个全新的变量跟原来的`str`并没有什么关系。

    **我们再来看一个例子：**
```JavaScript
var name = 'zhangsan';
console.log(name); // 输出zhangsan
name = 'lisi';
console.log(name); // 输出lisi
```
在这个例子中看起来`name`的值由“zhangsan”被修改为“lisi”，但其实并不是修改，原来的“zhangsan”还是“zhangsan”，只是被一个新的值“lisi”覆盖掉了而已。

    *这么说起来可能不是很形象，举个例子吧。现实生活中我们在画画的时候如果原本的颜色是红色的，但是我们想要将其修改为绿色的话我们是将新的绿色的颜料直接涂抹在原来红色的地方上面，这样在我们看来好像红色被修改为绿色，但红色还是红色只是被覆盖了我们看不到了而已，我们看到的是绿色。在这个例子中也是同理。*

# 四、五种基本数据类型详解
## typeof运算符
我们要检测一种基本数据类型时可以使用typeof运算符去检测，其语法是`typeof 变量`。
```JavaScript
console.log(typeof 123);            // 输出number
console.log(typeof 'string');       // 输出string
console.log(typeof true);           // 输出boolean
console.log(typeof null);           // 输出Object
console.log(typeof undefined);      // 输出undefined
console.log(typeof function(){});   // 输出function
```
> 注意：检测null时返回的是一个Object类型，这是因为null类型其实是一个空的对象应用，也就是一个对象但什么都没有。

## Number(数值类型)
在一些强语言中，数值类型可能会分为整数和浮点数，再根据占用的长度去再往下继续分为好几种数据类型，比如Java和C。但是在js中数值类型只有一种就是Number，在定义一个数值类型时加不加小数点都是可以的。
```JavaScript
var number = 1;
console.log(typeof number); // 输出number
```
上面的这个例子中为我们展示了如何去定义一个number类型的变量，以及如何去检测一个number变量。
js中如果需要表示一个非常大或者是非常小的数字可以使用科学计数法，例如：
```JavaScript
var num1 = 123e+5;  // 表示的是 12300000
var num2 = 123e-5;  // 表示的是 0.00123
```
js中表示最大的数值是`Number.MAX_VALUE`，与之对应的是`Numver.MIN_VALUE`表示的是js中最小的数值。

js中还有一个非常特殊的数值类型，就是NaN(Not A Number)即非数值，这个数值用于表示一个本来要返回数值的操作数未返回数值的情况（这样就不会抛出错误了）。例如，在其他编程语言中，任何数值除以0都会导致错误，从而停止代码执行。但在JavaScript中，任何数值除以0会返回NaN，因此不会影响其他代码的执行。

NaN有两个非同寻常的点，一个是任何涉及NaN的操作都会返回NaN，例如：
```JavaScript
console.log(NaN + 1); // 输出NaN
console.log(NaN - 1); // 输出NaN
console.log(NaN * 1); // 输出NaN
console.log(NaN / 1); // 输出NaN
```
> 注意：这个特点在多步计算中有可能导致问题，需要特别注意

另一个是NaN与任何值都不相等，包括NaN本身，例如：
```js
console.log(NaN == NaN); // 输出false
```

## String(字符串类型)
在js中的字符串类型是需要使用引号引起来的，使用单引号''或者是双引号""都可以，但是不要混合着用，也不能嵌套使用单双引号。

string类型有些特殊，因为字符串具有可变的大小，所以显然它不能被直接存储在具有固定大小的变量中。由于效率的原因，我们希望JS只复制对字符串的引用，而不是字符串的内容。但是另一方面，字符串在许多方面都和基本类型的表现相似，而字符串是不可变的这一事实（即没法改变一个字符串值的内容），因此可以将字符串看成**行为与基本类型相似的不可变引用类型**

如果需要在String类型中输出一些特殊的字符的话可以使用`\`字符来转义，例如：
```JavaScript
console.log("我说：\"我想吃西瓜\""); // 输出 我说："我想吃西瓜"
```
> 注意：如果想输出一个`\`的话则需要再使用一个`\`来转义，即`\\`

## Boolean(布尔类型)
Boolean类型只有两个值：true表示真和false表示假。
虽然Boolean类型的字面值只有两个，但JavaScript中所有类型的值都有与这两个Boolean值等价的值。要将一个值转换为其对应的Boolean值，可以调用类型转换函数Boolean()，例如：
```js
var message = "Hello World";
var messageAsBoolean = Boolean(message);
```
在这个例子中，字符串message被转换成了一个Boolean值，该值被保存在messageAsBoolean变量中。可以对任何数据类型的值调用Boolean()函数，而且总会返回一个Boolean值。至于返回的这个值是true还是false，取决于要转换值的数据类型及其实际值。下表给出了各种数据类型及其对象的转换规则。

| 数据类型    |  转换成true值 | 转换成false值 |
| :-------- | -------:| :--: |
| Boolean | true | false |
| String | 任何非空字符串 | ""空字符串 |
| Number | 任何非0数值（包括无穷大） | 0和NaN |
| Object | 任何非空对象 | null |
| Undefined | 不适用 | Undefined |  

**ps：如果需要将一个变量转换成对应的布尔值的话，可以使用`!!`(两个感叹号)操作符转换。例如：**
```js
var message = "Hello World";
console.log(!message);  // 输出false
console.log(!!message); // 输出true
```
使用一个`!`代表的是取反的操作，即转换为Boolean值之后取的是相反的值，但是再使用多一个`!`之后呢再将取反的操作再次取反之后就可以得到原来对应的Boolean值

## Null(空引用类型)
Null类型是只有一个值的特殊数据类型，这个特殊的值就是`null`代表的是一个空对象，一个空引用。从逻辑角度来看，`null`值表示一个空对象指针，而这也正是使用`typeof`操作符检测`null`时会返回`object`的原因。

如果定义的变量准备在将来用于保存对象，那么最好将该变量初始化为`null`而不是其他值。这样一来，只要直接检测`null`值就可以知道相应的变量是否已经保存了一个对象的引用了。例如：
```js
var obj = null;
if (obj != null) {
    // 对obj对象进行的操作
}
```

## Undefined(未定义)
Undefined类型只有一个值，即特殊的`undefined`。在使用`var`声明变量但未对其加以初始化时，这个变量的值就是`undefined`。例如：
```js
var a;
console.log(a); // 输出undefined
```

# 五、引用数据类型
## 引用数据类型的介绍
除了上面介绍的基本数据类型之外，其他的就是引用数据类型了。引用类型统称是Objcet，所有的引用数据类型的数据都是继承自Object的。例如：Array（数组）、Date（日期）、Function（函数）等。这些都是引用数据类型，也都是继承自Object。

## 引用数据类型的特点
1. **引用类型的值是可变的**

    引用数据类型跟基本数据类型最大的不同就是内存存放位置的不同，上文提过基本数据类型是存放在**栈内存**中的，引用数据类型则是存放在**堆内存**中的。引用数据类型将堆内存中的地址存放在栈内存中，通过变量标识符去取到地址再进行访问。而堆内存的大小并不是固定的，所以引用数据类型的值是可变的。基于这个特点，我们如果想要在一个引用数据类型中取增加或者是删除一些内容的话也都是可以实现的，我们甚至可以在一个引用中去定义另一个引用，我们来看一个例子：
    ```js
    var obj = {name:"zhangsan"};
    console.log(obj.name);  // 输出zhangsan
    obj.name = "lisi";
    console.log(obj.name);  // 输出lisi
    obj.age = 18;
    console.log(obj.age);   // 输出18
    obj.age = null;
    console.log(obj.age);   // 输出null
    obj.printObj = function () {
        console.log(this.name + "---" + this.age);
    }
    obj.printObj(); // 输出lisi---null
    ```

2. **引用数据类型的比较是引用的比较**

    我们先来看一个例子：
```js
var obj1 = {};
var obj2 = {};
console.log(obj1 == obj2);   // 输出false
console.log(obj1 === obj2);  // 输出false
```
在这个例子中，`obj1`和`obj2`都是定义了相同的内容，都是`{}`，但是在比较的时候js却不认为这是两个相同的对象。原因在于声明这两个对象时，会在堆内存中开辟出两块不同的空间去分别存放这两个变量，然后将堆内存存放这两个对象对应的内存地址再去存在栈内存中去。在比较时并不会去比较两个对象具体的区别，而是在栈内存中将地址取出来之后进行比较，如果地址相同都是指向同一块内存的话才会认为是相同的对象，反之则认为是不同的对象。

    我们再来看一个例子：
```js
var obj1 = {name:"zhangsan"};
var obj2 = obj1;
console.log(obj1.name); // 输出zhangsan
console.log(obj2.name); // 输出zhangsan
obj2.name = "lisi";
console.log(obj1.name); // 输出lisi
console.log(obj2.name); // 输出lisi
```
在这个例子中，首先声明了一个对象`obj1`再将`obj1`的值赋给`obj2`，这时其实是将`obj1`的地址赋给了`obj2`，也就是说在栈内存中的`obj1`和`obj2`的值是相等的，都是保存的相同的地址，指向的是同一块堆内存中的空间。所以不管是改变了`obj1`还是改变了`obj2`其实都是在操作的同一块内存空间，做出来的改变也是相同的，另一个指向这个空间的对象也是会跟随着一起改变。

3. **instanceof**
一个对象如果我们需要去检测它是不是属于某个对象的实例的话，我们需要使用`instanceof`关键字，语法是`实例 instanceof 对象`表示这个实例是否属于这个对象。例如：
```js
console.log(({}) instanceof Object);    // 输出true
console.log(([]) instanceof Array);     // 输出true
```