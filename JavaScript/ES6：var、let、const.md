---
title: ES6：var、let、const
abbrlink: 37909
date: 2020-06-15 20:40:22
categories:
 - JavaScript
 - ES6
---
> let 这个关键字都是在 ES6 中新出现的，作用与 var 是一样的，可以用来声明变量。const 关键字也是在 ES6 中新出现的，不同的是 const 是用来声明常量。

# let & var

#### 块级作用域
「块级作用域」这个概念是在 ES6 中新引入的概念，就是一个 `{}` 「花括号」而已。所有的花括号之内都是一个单独的块级作用域，但是有一种特殊情况不是，那就是在声明对象的时候字面式声明一个对象的时候不是。

#### let和var的区别
1. let 和 var 最大的区别就是 **let声明的对象只在当前作用域生效**
例如：
```js
{
    var a = 1
    let b = 2
}
console.log(a) // 1
console.log(b) // 报错
```

2. 不能重复声明
```js
var a = 10
console.log(a)
var a = 20
console.log(a)

let b = 10
console.log(b)
let b = 20 
console.log(b)
```
上面的代码将会直接报错

3. let 不存在「变量提升」
使用 var 关键字声明的变量在执行上下文中会有一个变量提升的现象，但是使用 let 声明的变量将不会出现这个现象，比如：
```js
console.log(a)  // undefined
var a = 10

console.log(b)  // 报错
let b = 10
```

4. 暂存死区
```js
let a = 10
{
    console.log(a)
    let a = 20
}

{
    let a = 20
    console.log(a) // 20
}
```
上面的这段代码将会直接报错。因为 ES6 中单独的作用域内使用 let 或者是 const 声明的变量将会形成一个封闭的作用域，这直接导致重名的变量无法访问到父级作用域的变量，进而报错。

# const 常量
1. 声明常量
const 声明时与 let 声明并没有什么区别，唯一的区别是使用 const 声明常量时需要在声明的同时初始化。
```js
const a = 10
const a // 报错
```

2. 不能修改
声明的既然是常量，那么当然是不允许被修改的。例如：
```js
const a = 10
a = 20 // 报错
```
    **如果声明的是一个引用类型的数据，那么它将可以被修改**
```js
const obj = {
    name: 'zhangsan'
}
console.log(obj.name)  // zhangsan

obj.name = 'lisi'
console.log(obj.name)  // lisi
```
实际上 const 声明的常量仅仅只是「锁定」了**栈内存**中的值，而引用数据类型实际存放的地方是在**堆内存**中的。所以对象类型的值是可以被修改的。

    那么我们既然是声明了一个常量的话当然是不希望这个常量能够被修改，如果是一个对象类型的常量的话，我们必须通过 `Object.freeze()` 这个方法来「冻结」这个对象类型的常量。
```js
const obj = {
    name: 'zhangsan'
}
console.log(obj.name) // zhangsan

// 冻结
Object.freeze(obj)

obj.name = 'lisi'
console.log(obj.name) // zhangsan
```
在上面这个例子中两次的输出都是 `zhangsan` 这说明了我们并没有修改到 `obj` 这个对象，这样子就是声明了一个对象类型的常量表了。