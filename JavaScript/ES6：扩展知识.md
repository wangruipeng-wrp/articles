---
title: ES6：扩展知识
abbrlink: 18551
date: 2020-06-24 22:40:58
categories:
 - JavaScript
 - ES6
---

> 本文将介绍一些关于ES6的比较基础的比较零碎的知识。
<!-- more -->

# 模板字符串
模板字符串在字符串拼接方面比起以前要方便很多。模板字符串的语法是一对反引号 ( `` ) 
```js
var obj = {
    name : 'zhangsan',
    age : 20,

    say1() {
        console.log('我叫 ' + this.name + '，今年 ' + this.age + ' 岁了');
    },

    say2() {
        console.log(`我叫 ${ this.name }，今年 ${ this.age } 岁了`)
    }
}

obj.say1();
obj.say2();

// 以上 say1 和 say2 方法打印的效果完全相同
```
顺带说一下 ES6 中一个比较有用的字符串的方法 includes 方法。判断字符串中是否包含了其他字符串
```js
let str = 'Hello World!';

console.log(str.includes('o W'));  // true
```

---

# for-of 循环
for-of 循环类似于 Java 中的 foreach 循环。来看代码：
```js
let arr = ['zhangsan', 'lisi', 'wangwu'];
for (let name of arr) {
    console.log(name)
}
// zhangsan lisi wangwu
```
也可以用来遍历字符串
```js
let str = 'Hello World!';
for (let i of str) {
    console.log(i);
}
// Hello World!
```

---

# unicode表示法
ES6 之前 unicode 码点仅可以表示的范围是（0000 - ffff）超出这个范围的将无法表示例如 emoji 表情。ES6 中表示 unicode 的语法是 `\u{码点}`，这样就可以表示超出范围的码点。
```js
console.log('\u{1f436}'); // 🐶
```

---

# 箭头函数
箭头函数的标志是一个箭头 `=>` 就是这个样子，箭头函数可以简化函数的书写过程，使得代码更加简洁。
```js
const add1 = (a, b) => a + b;
const add2 = function(a, b) {
    return a + b;
}

add1(2, 2);
add2(2, 2);

// add1 和 add2 执行的效果相同
```
> 小提示：如果只有一个参数的情况下是可以不用加括弧的哦

上面的代码中仅仅只是执行一行代码而已，如果需要执行多行代码，那么需要像下面这样

```js
const add1 = (a, b) => {
    a += 1;
    return a + b;
};
const add2 = function(a, b) {
    a += 1;
    return a + b;
}

add1(2, 2);
add2(2, 2);

// 同样，add1 和 add2 执行效果相同
```

> 小技巧：如果我们需要执行一个有返回值的函数但是又不需要函数的返回值，可以在箭头函数的函数体前加 void
```js
let pop = arr => void arr.pop();

let arr = [1, 2, 3];
console.log(pop(arr));  // undefined
console.log(arr);  // 1,2 
```

## 区别普通函数

1. 没有 arguments 对象
```js
let log = () => {
    console.log(arguments);
}

log(1, 2, 3);
// 报错：arguments is not defined
```
小技巧：如果需要使用 arguments 对象可以使用 `...参数名` 的方式来代替，这里的 `...` 指的是函数的剩余参数。
```js
let log = (...args) => {
    console.log(args);
}

log(1, 2, 3);
// 输出 [1, 2, 3]
```

2. 没有专属的 this 对象
```js
const obj = {
    name: 'zhangsan',
    say1() {
        console.log(this);
    },
    say2: () => {
        console.log(this);
    }
}

obj.say1(); // obj 对象
obj.say2(); // window 对象
```
> 可以将箭头函数的 this 理解为上一级环境中的 this

    小技巧：在开发中经常需要去在回调函数中去调用上一级的 this 对象，在这种情况下我们要先将上一级的 this 对象保留下来，例如：
```js
const obj = {
    name: 'zhangsan',
    age: null,
    getAge() {
        let _this = this;

        // 使用计时器模拟回调函数
        setTimeout(function() {
            _this.age = 20;
            console.log(_this.age);
        }, 1000);
    }
}

obj.getAge(); // 20
```
上面的代码如果使用箭头函数可以省略掉 `let _this = this` 的这一步
```js
const obj = {
    name: 'zhangsan',
    age: null,
    getAge() {
        // 使用计时器模拟回调函数
        setTimeout( () => {
            this.age = 20;
            console.log(this.age);
        }, 1000)
    }
}

obj.getAge(); // 20
```
---
# 对象的简洁表示法
```js
// 不使用简洁表示法
const getUserInfo = () => {
    const name = 'zhangsan',
    const age = 20,
    return {
        name: name,
        age: age,
        say: function() {
            console.log(this.name + this.age);
        }
    }
}

// 使用简洁表示法
const getUserInfo = () => {
    const name = 'zhangsan',
    const age = 20,
    return {
        name,
        age,
        say() {
            console.log(this.name + this.age);
        }
    }
}
```
---
# 对象的新方法和新属性
## Object.is
> 用来判断两个对象是否相同，与之前的判断主要有以下两个区别
```js
console.log(Object.is(+0, -0));     // false
console.log(+0 === -0);             // true

console.log(Object.is(NaN, NaN));   // true
console.log(NaN === NaN);           // false
```

## Object.assign
> 用来合并对象。注意：合并时仅仅是浅拷贝合并，也就是仅拷贝对象在栈内存中的地址。
```js
const obj = Object.assign({a:1}, {b:2}, {c:3});
console.log(obj);  // {a:1, b:2, c:3}

// 浅拷贝例子：
let obj1 = {a: 1}
let obj2 = Object.assign(obj1, {b:2})

obj2.a = 100;

console.log(obj1);  // {a:100, b:2}
```

## Object.keys、Object.values、Object.entries
> 用来取出对象中的键和值
```js
const obj = {
    a: 1,
    b: 2,
    c: 3,
    d: 4,
    e: 5,
}

console.log(Object.keys(obj));      // ['a','b','c','d','e']
console.log(Object.values(obj));    // [1,2,3,4,5]
console.log(Object.entries(obj));   // [['a',1], ['b',2], ['c',3], ['d',4], ['e',5]]
```