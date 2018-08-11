---
layout:     post
title:      "getter/setter和其他对象属性特性"
subtitle:   "Getter/Setter and Other Object Property's Attributes"
date:       2018-08-07 12:00:00
author:     "Sandii"
header-img: "img/pexels/010.jpg"
catalog: true 
tags:
    - vue
---

很多年以后，面对Vue源码里各种眼花缭乱的技巧，我回想起了在中学球场上被放暑假回乡的CUBA大神们碾压那个遥远的下午。练习赛后，教练说可能你们只看到了很炫的球技，没有看到这些球技背后**对基本功日复一日持之以恒的磨练**。你们身体素质本来就没有别人好，日常训练量连他们的一半都不到，怎么可能打得过呢。

说实话，Vue的源码里使用的很多js特性，我都很不熟悉。智力本身就比别人差了很远，基本功还不扎实，这怎么行呢。十几年后再次明白了这个道理，惭愧得无地自容。无论是多年前的篮球，还是现在的编码，我都缺少了对基本功日复一日持之以恒的磨练。

以上是检讨。现在开始今天的课题：复习一下getter/setter以及其他几个属性特性。它们是Vue响应式数据的力量来源。

## 数据属性 vs. 访问器属性

按照`javascript definitive guide`的6.6中介绍，对象的属性可分为两类：
1. data property 数据属性（常见）
1. accessor property 访问器属性

```
const o = {
    // 数据属性很常见
    dataProp : 1,

    // 访问器属性由getter和setter函数定义
    get accessorProp () {},
    set accessorProp (v) {},
};
```

访问器函数的使用规则是：
- getter函数在每次取值时调用，取值就是函数的返回值
- setter函数在每次设值时调用
- 函数中可以用this访问本对象

getter和setter可以只写一个：
- 如果只有getter，属性可读不可写（这是声明只读属性的常用做法）
- 如果只有setter，属性可写不可读

另外还要注意递归调用的问题：

```
// 错误写法！将导致无限循环调用，最后内存溢出。
const o = {
    get a () { return this.a * 2; },
    set a (v) { this.a = v; },    
};

// 正确写法！用一个其他属性作为中转站。以$开头说明是一个内部属性。
const o = {
    get a () { return this.$temp * 2 || 0; },
    set a (v) { this.$temp = v; },
    // 这里使用了temp属性作为中转
};
```

## 其他属性特性

在ES3中，属性除了属性名`name`和属性值`value`之外，还有另外三个特性（attribute）：
- writable 可写
- enumerable 可枚举
- configurable 可配置
但是ES3没有提供修改这三个特性的API，因此所有属性都是可写可枚举可配置的。

ES5开始，提供了修改属性特性的API，并且引入了两个访问函数`getter`和`setter`。归纳起来：除了name之外，两类属性各有4个特性。其中2个相同，2个不同。

|数据属性|访问器属性|
|-|-|
|value|getter|
|writable|setter|
|enumerable|enumerable|
|configurable|configurable|

`Object.defineProperty(obj, propName, propDesc)`可以用于设置和修改属性特性。特性不用全都写：
- 新设置的话不写认为是false或undefined
- 修改的话不写认为是不变

```
// 声明一个连prototype都没有的空对象
const o = Object.create(null);

// 声明一个数据属性
Object.defineProperty(o, 'a', {
    value : 0,
    writable : true,
    enumerable : true,
    configurable : true,
});

// 声明一个访问器属性
Object.defineProperty(o, 'b', {
    get () { return this.$temp || 0; },
    set (v) { this.$temp = v; },
    enumerable : true,
    configurable : true,
});

// 报错！
// Uncaught TypeError: Invalid property descriptor. 
// Cannot both specify accessors and a value or writable attribute.
// 编译器不知道你想声明数据属性还是访问器属性
Object.defineProperty(o, 'c', {
    get () { return this.$temp || 0; },
    set (v) { this.$temp = v; },
    value : 0,
    writable : true,
    enumerable : true,
    configurable : true,
});
```

## 操作属性特性API

除了`Object.defineProperty`之外还有其他用于操作属性特性的API。

```
// 1. `Object.getOwnPropertyDescriptor(obj, propName)` 
// 用于获取属性特性
// {value : 1, writable : true, configurable : true, enumerable : true }
Object.getOwnPropertyDescriptor({a : 1}, 'a'); 
Object.getOwnPropertyDescriptor({}, 'a');  // undefined 不存在
Object.getOwnPropertyDescriptor({}, 'toString');  // undefined 无法获取继承属性

// 2. `Object.defineProperties(obj, { propName ： propDesc })` 
// 批量设置
// 注意返回值是目标对象，单个设置也是这样
const obj = Object.defineProperties({}, {
    a : {
        value : 1,
        writable : true,
        enumerable : true,
        configurable : true,
    },
    b : {
        get () { return this.$temp; },
        set (v) { this.$temp = v; },
        enumerable : true,
        configurable : true,
    },
});

// 3. `Object.create(prototype, { propName ： propDesc })` 用法和Object.defineProperties类似
// ...
```

## 只读属性

```
// 1. 先声明两个只读属性
const o = Object.defineProperties({}, {
    a : {
        get () { return 1; },
        enumerable : true,
        configurable : true,
    },
    b : {
        value : 1,
        writable : false,
        enumerable : true,
        configurable : true,
    },
});
// 验证一下确实是只读的
o.a = 2;
console.log(o.a); // 1
o.b = 2;
console.log(o.b); // 1


// 2. 但还可以这么修改它的值
Object.defineProperties(o, {
    a : { get () { return 2 } },
    b : { value : 2},
});
// 修改成功
console.log(o.a); // 2
console.log(o.b); // 2

// 3. 那么必须把它们锁死
Object.defineProperties(o, {
    a : { configurable : false },
    b : { configurable : false },
});
// 这下才是真正的只读属性
Object.defineProperties(o, {
    a : { get () { return 3 } }, // TypeError
    b : { value : 3 },  // TypeError
});
console.log(o.a); // 2
console.log(o.b); // 2
```
