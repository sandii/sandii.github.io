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

最近的业余时间在看Vue的源码。Vue数据响应式力量的来源就是getter/setter。

getter和setter属于js中比较底层的`元编程`(meta programming)API，所谓元编程就是修改编程语言。而getter和setter用于修改对象属性的读写。具体情况看个例子就知道了：

```
// 对象属性的基本读写，没毛病
const a = { x : 1 };
console.log(a.x); // 1
a.x = 2;
console.log(a.x); // 2

// 使用getter和setter修改属性读写
const b = {
  get x () { return 1; },  // 读：返回值就是读取到的值
  set x () { console.log('haha'); },
};
console.log(b.x); // 1
b.x = 2; // haha
console.log(b.x); // 1
```
现在应该很明白了。getter和setter可以介入js对象的正常读写，改变其正常的行为。我们至少可以利用他们来实现两种功能：
1. 监听数据变化。就像上面的例子中，一旦有人试图修改属性就打印haha。Vue也是这么做的，当然肯定会比打印haha更复杂一些。
2. 设置只读属性。


## 数据属性 vs. 访问器属性

**js对象的属性可分为两类**（犀牛书6.6）。比如从上面的例子可以发现，我们声明a.x和b.x的方式是完全不同的，他们分别是：
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


## 不严格的只读属性

现在我们开始考虑一下只读属性的问题。上面的例子中的其实就已经声明了一个只读属性b.x，因为负责写值的setter函数除了打印haha啥也没做。所以，要声明一个只读属性很简单，只写一个getter就行了：

```
const c = {
    get x () { return 1; },
};
```

但实际上这么写并不严格，还有办法修改c.x。这就需要了解一下属性的特性了(attributes of property)和设置它们的API。


## 属性特性

在ES3中，属性除了属性名`name`和属性值`value`之外，还有另外三个特性（attribute）：
- writable 可写
- enumerable 可枚举
- configurable 可配置

但是ES3没有提供修改这三个特性的API，因此所有属性都是可写可枚举可配置的。而ES5开始，提供了修改属性特性的API。最常用的是`Object.defineProperty`。我们可以这么设置一个数据属性的特性:

```
Object.defineProperty(o, 'a', {
    value : 0,
    writable : true,
    enumerable : true,
    configurable : true,
});
```

再看访问器属性，`getter`和`setter`分别对应数据属性的`value`和`writable`，负责属性的读写，因此完整地总结起来属性的特性应该是这样的：

|数据属性|访问器属性|
|-|-|
|value|getter|
|writable|setter|
|enumerable|enumerable|
|configurable|configurable|

我们分别设置一个数据属性和一个访问器属性试试：

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

// 试试混合设置
Object.defineProperty(o, 'c', {
    get () { return this.$temp || 0; },
    set (v) { this.$temp = v; },
    value : 0,
    writable : true,
    enumerable : true,
    configurable : true,
});

// 无情地报错了！
// Uncaught TypeError: Invalid property descriptor. 
// Cannot both specify accessors and a value or writable attribute.
// 编译器不知道你想声明数据属性还是访问器属性
```

需要注意的是设置和修改属性特性时，其实**特性不用全都写**：
- 新设置的话不写认为是false或undefined
- 修改的话不写认为是不变


## 操作属性特性API

除了`Object.defineProperty`之外还有其他操作属性特性的API。

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

把概念和API都搞清楚之后，只读属性的问题关键就很清楚了：`configurable`。只要把设为false，才是真正彻底锁死了属性的值。

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


## 常见陷阱：递归调用

最后我们来看一下初学者最常见的坑：递归调用。我们声明一个访问器属性，想要每次读取的时候得到真实值的两倍：

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

## 总结

- getter和setter是Vue数据响应式的底层原理
- getter和setter属于`元编程`，用于修改js对象属性的读写行为
- 对象的属性分为两类，**数据属性**（常见）和**访问器属性**（getter/setter）
- 对象的属性有4个特性 `value/getter` `writable/setter` `enumerable` `configurable`
- 操作属性的特性的API有4个 `Object.defineProperty` `Object.defineProperties` `Object.create` 和 `Object.getOwnPropertyDescriptor`
- 不严格的只读属性只需要把writable设为false或不写setter就行了
- 严格的只读属性还需要把configurable设为false
- 使用getter和setter时要注意避免递归调用的问题，常见的做法是使用其他变量作为中转


## 背单词

|英文|翻译|说明|
|-|-|-|
|meta programming|元编程|修改编程语言本身的特性|
|data property/accessor property|数据属性/访问器属性|js对象的属性分为这两类|
|attributes of property|属性特性|-|
