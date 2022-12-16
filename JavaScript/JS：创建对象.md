---
title: JavaScript：创建对象
abbrlink: 15655
date: 2020-06-08 21:12:26
categories:
 - JavaScript
---
这篇博客将会记录一下js中创建对象的几种不同的方式，分别是字面式声明、`new`关键字声明、构造函数声明、工厂模式声明、原型声明。
<!-- more -->

# 字面式声明
声明格式：
```js
// 直接实例化一个对象
var obj = {
    name: 'zhangsan',
    age: 18,
    
    run: function () { 
        return "姓名：" + this.name + "年龄：" + this.age;
    },
}

// 调用属性和函数
obj.name;
obj.age;
obj.run();
```
---

# new关键字声明
`Object`对象是所有对象的父类，也称为根类、基类、超类。js中的所有对象都是`Object`对象的延伸，都是`Object`的子类。

声明格式：
```js
var obj = new Object();
obj.属性 = 属性值;
obj.属性 = 属性值;
obj.函数 = function() {
    函数体
};
```
---

# 构造函数创建对象
```js
// 创建构造函数
function obj(name, age) {
    this.name = name;
    this.age = age;
    this.run = function () {
        return "姓名：" + this.name + "年龄：" + this.age;
    }
}

// 实例化
var obj = new obj();

// 调用属性和方法
obj.name;
obj.age;
obj.run();
```
---

# 工厂模式创建对象
```js
// 创建一个工厂
function createObj(name, age) {
    var obj = new Object();
    obj.name = name;
    obj.age = age;
    obj.run = function() {
        return "姓名：" + this.name + "年龄：" + this.age;
    }
    return obj;
}

// 通过工厂实例化对象
var obj = createObj('zhangsan', 18);

// 调用属性和函数
obj.name;
obj.age;
obj.run();
```
---

# 原型模式创建对象
概述：声明一个空的函数，再利用`prototype`去定义属性和函数。
```js
// 声明空函数
function fn() {}

// 利用prototype去定义属性和函数
fn.prototype.name = 'zhangsan';
fn.prototype.age = 18;
fn.prototype.run = function () {
    return "姓名：" + this.name + "年龄：" + this.age;
}

// 实例化
var obj = new fn();

// 调用属性和方法
obj.name;
obj.age;
obj.run();
```

第二种方式：直接使用一个json对象定义属性和函数
```js
// 声明空函数
function() {}

// 利用json对象定义属性和函数
fn.prototype = {
    name: 'zhangsan',
    age: 18,
    run: function() {
        return "姓名：" + this.name + "年龄：" + this.age;
    }
}

// 实例化
var obj = new fn();

// 调用属性和方法
obj.name;
obj.age;
obj.run();
```
---


# 混合模式
混合模式是**构造模式**混合了**原型模式**一起创建的对象。
```js
// 声明一个构造函数
function Obj(name, age) {
    this.name = name;
    this.age = age;
}

// 原型添加属性和方法
Obj.prototype = {
    run: function() {
        return "姓名：" + this.name + "年龄：" + this.age;
    }
}

// 实例化对象
var obj = new Obj('zhangsan', 18);

// 调用属性和方法
obj.name;
obj.age;
obj.run();
```