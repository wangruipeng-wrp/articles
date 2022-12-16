---
title: ES6：解构赋值
abbrlink: 53099
date: 2020-06-16 19:30:20
categories:
 - JavaScript
 - ES6
---

> 解构赋值语法是一个 JavaScript 表达式，这使得可以将 **值从数组** 或 **属性从对象** 提取到不同变量中。

<!-- more -->

# 数组的解构赋值
数组的解构赋值是有序的，将会按照顺序去匹配

#### 简单的解构赋值

``` js
const arr = [1, 2, 3, 4]
let [a, b, c, d] = arr

// 变量 a b c d 的值将会对应上数组中的 1 2 3 4
```
在 ES6 之前，「解构赋值」还没出现的时候，想要实现以上功能只能够一个个的去定义 `a b c d` 这四个变量。因此，「解构赋值」的出现就是为了简化一些代码的编写，实现更加高效的开发。

#### 数组嵌套的解构赋值

``` js
const arr = ['a', 'b', ['c', 'd', ['e', 'f''g']]]
let [, , [, , [, , g]]] = arr

// 变量 g 将对应 arr 中的字符 g
```

#### 扩展运算符： `...` 

``` js
const arr1 = [1, 2, 3]
const arr2 = [4, 5, 6]
const arr3 = [7, 8, 9]

let arr4 = [...arr1, ...arr2, ...arr3]

// arr4 = [1, 2, 3, 4, 5, 6, 7, 8, 9]
// 注意：arr4 将是一个一维数组而不是二维数组
```

``` js
const arr = [1, 2, 3, 4]
let [a, b, ...c] = arr

// 变量 c 将对应数组中 c 之后的所有值组成的新数组
// 注意：这种写法扩展运算符必须放在最后一位
```

#### 默认值

``` js
const arr = [1, undefined, 3]
let [a, b = 2, c, d] = arr

// 变量 a b c d 将对应成[1, 2, 3, undefined]
// 没有匹配到的值将会默认为 undefined，匹配到 undefined 将会自动赋为默认值
```

#### 数组解构赋值的使用

1. **交换变量**
``` js
let a = 20;
let b = 10;

[a, b] = [b, a];

// a = 10
// b = 20
```

2. **接收函数中的数组返回值**
``` js
function getUserInfo() {
    return [
        true,
        {
            name: 'zhangsan',
            age: 18
        },
        '请求成功'
    ]
}

let [status, data, msg] = getUserInfo()
```
--- 

# 对象的解构赋值
对象的解构赋值是无序的，将直接按照属性名称去匹配

#### 简单的解构赋值
```js
let obj = {name: 'zhangsan', age: 18}
let {name, age} = obj

// name : 'zhangsan'
// age : 18
// 按照属性名称匹配对应的值
```

#### 对象与数组嵌套的解构赋值
```js
let obj = {
    name: 'zhangsan',
    age: 18,
    hobby: [{
        hobbyName: '听歌'
    },
    {
        hobbyName: '看电影'
    }]
}

let {
    hobby:
    [
        { hobbyName: name1 },
        { hobbyName: name2 }
    ]
} = obj

// name1: '听歌'
// name2: '看电影'
```
注意：`hobby` 后的“:”跟着的是 `obj` 对象中的 `hobby` 数组，`hobbyName` 后的“:”跟着的是为了区分两个同名的 `hobbyName` 所起的别名

#### 扩展运算符：`...`
```js
let obj = {
    name: 'zhangsan',
    age: 18,
    sex: '男'
}
let {name, ...obj1} = obj;

// obj 对象中的 age 和 sex 属性将会被封装成 obj1 对象
// 注意：这种方式使用的扩展运算符只能够放在最后一位
```
```js
let obj1 = {
    name: 'zhangsan'
}
let obj2 = {
    age: 18
}
let obj3 = {
    sex: '男'
}

let obj = {...obj1, ...obj2, ...obj3}

// 对象 obj 中会有 obj1-3 的全部属性
// 注意：如果存在重名属性，后面的会覆盖掉前面的
```

#### 对已经声明的变量进行解构赋值
```js
let age;
let obj = {
    name: 'zhangsan',
    age: 18
};
({ age } = obj)

// 注意：以上的解构赋值语句如果没有加 () 会被视为是一个作用域而报错
```

#### 默认值
```js
let obj = {
    name: 'zhangsan',
    age: undefined
}
let {name, age = 18, sex = '男'} = obj

// 在匹配时，如果匹配不到对应的属性或者是对应的属性为 undefined 时，将会寻找默认值自动匹配
```

#### 对象解构赋值的使用

1. **传递乱序参数和设置默认值**
```js
// 不使用解构赋值
function AJAX(option) {
    $.ajax({
        url: option.url,
        type: option.type || 'get',
        data: option.data
    })
}

// 使用解构赋值
function AJAX( { url, data, type = 'get' } ) {
    $.ajax({
        url: url,
        type: type,
        data: data
    })
};

AJAX({
    url: '/getUserInfo',
    data: {
        id: 1,
    }
});
```

2. **获取一个函数中的多个返回值**
```js
function getUserInfo (userId) {
    // ...ajax
    return {
        status: true,
        data: {
            name: 'zhangsan',
        },
        msg: '请求成功'
    };
};

let {status, data, msg} = getUserInfo(123);
```

# 字符串的解构赋值
1. **直接取值**
```js
let str = "Hello,World!";
let [a, b, c] = str;

// a b c 三个变量将会分别对应字符串 str 的前三个字符
```

2. **扩展运算符：`...`**
```js
let str = "Hello,World!";
let [a, b, ...c] = str;

// 变量 a b 依旧对应前两个字符，变量 c 则对应的是后面的每个字符所组成的数组
```

3. **分割字符串**
```js
let str = "Hello,World!";
let [ ...spStr1 ] = str;

// 上面的代码效果等同于：let spStr1 = str.split('');
```

4. **提取属性**
```js
let str = "Hello,World!";
let { length } = str;

// length = 12
```