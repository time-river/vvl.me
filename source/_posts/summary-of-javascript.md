---
title: JavaScript 小结
date: 2016-07-21 18:00:43
categories: style
tags: JavaScript
---

再识 JavaScript，对[廖雪峰教程](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)的一些总结。

## 数据类型和变量

* Number
  * 整数
  * 浮点数
  * `NaN` -- Not a number，无法计算的结果用它表示
  * `Infinity` -- 无穷大
* 字符串
  * `'abc'`等价于`"abc"`，`''`或`""`本身只是一种表示方式
* 布尔值
  * `true`
  * `false`
  * 布尔运算 --  `&&` `||` `!`
* 比较运算符
  * `>` `>=` `<` `<=` `==` `===`
  * `==` -- 自动转换数据类型再比较
  * `===` -- 不会自动转换数据类型，若数据类型不一致，返回`false`，若一致则再比较
  * 建议使用`===`
  * `NaN == NaN // false` -- `NaN`与所有值都不相等，包括自己。判断`NaN`方法是通过`isNaN()`函数
  * `1 / 3 === (1 - 2 / 3); // false` -- 比较两浮点数是否相等，只能计算他们只差的绝对值 -- `Math.abs(1 / 3 - (1 - 2 / 3)) < 0.0000001; // true`
  * `null` & `undefined`
    * `null`表示一个“空”的值，`0`是一个数值，`''`表示长度为0的字符串
    * `undefined`表示未定义。
* 数组
  * 创建数组的方法是通过`Array()`函数实现 -- `new Array(1, 2, 3)`
  * 出于代码的可读性，建议直接使用`[]`
  * 数组元素可以通过索引访问
* 对象
  * 是一组由键-值组成的无序集合
  * 键都是字符串类型，值可以是任意数据类型
  * 获取一个对象的属性，用`对象变量.属性名`的方式
* 变量
  * 变量名是大小写英文、数字、`$`和`_`的组合，且不能用数字开头，变量名野不能是 JavaScript 的关键字
  * 申明一个变量用`var`语句
* strict 模式
  * 启用 strict 模式的方法是在 JavaScript 代码的第一行写上 `'use strict'`
  * 此模式强制通过`var`申明变量

## 对象

* 字符串对象
  * `var s = 'I\'m \"OK\"!';` -- `\`转义
  * `s.length`            -- 字符串长度
  * `s[0]`                -- 索引访问
  * `s.toUpperCase()`     -- 把一个字符串全部变为大写
  * `s.toLowerCase()`     -- 小写
  * `s.indexOf("OK")`     -- 搜索指定字符串出现的位置，0开始，没有返回-1
  * `s.substring(0, 2)`   -- 返回指定索引区间的子串
* 数组
  * `var arr = [1, 2, 3.14, 'Hello', null, true];` -- 包含任意数据类型
  * `arr.length`
  * `arr[0] = 'x'`                                 -- 索引访问并修改
  * `arr.indexOf(1)`
  * `arr.slice(0, 3)`                              -- 对应 _String_ 的 `substring()`
  * `arr.push('A', 'B')` `arr.pop()`               -- 数组末尾
  * `arr.unshift('A', 'B')` `arr.shift()`          -- 数组头部
  * `arr.sort()`
  * `arr.reverse()`
  * `arr.splice(0, 1, 3, 0)`                       -- 从指定索引位置开始删除若干元素，再从该位置添加若干元素
  * `arr.concat(["A", "B", "C"])`                  -- 把当前的数组与另几个元素/数组连接起来，返回新数组
  * `arr.join('-')`
* 多维数组
* 对象
  * 无序的集合数据类型，由若干键值对组成
  * `{...}`表示对象，`xxx:xxx`申明键值对，`,`隔开键值对，`,`不加再末尾
  * `.`或`[]`访问属性
  * 可自由给一个对象添加或删除属性
  * 属性检测使用`in`操作符

## 分支跳转

```javascript
if () {
  ...
} else {
  ...
}

// 经典 for 循环
for (var i=0; i<arr.length; i++) {
  ...
}
// for ... in
for (var key in arr) {
  ...
}
// iterable类型
for (var x of a) {
  ...
}

while (n > 0){
  ...
}
do {
  ...
} while (n < 100);
```

## Map / Set / Iterable

* Map -- 键值对结构
  * `var m = new Map([['Michael', 95], ['Bob', 75], ['Tracy', 85]]);`
  * `m.set('Adam', 67);`
  * `m.has('Adam');`
  * `m.get('Adam');`
  * `m.delete('Adam');`
* Set -- 集合
  * `var s = new Set([1, 2, 3]);`
  * `s.add(4);`
  * `s.delete(3);`
  * `s.has(2);`
* Iterable
  * 可使用`for ... of`循环，此循环只循环集合本身的元素
  * 内置`forEach(function (element, index, array))`方法，接受一个函数，每次迭代自动回调该函数
  * _element_ -- 当前元素本身；_index_ -- 当前元素索引；_array_ -- 当前的对象

## 函数

* 定义
  * `function abs(x) { ... }`
  * 允许传入任意参数而不影响调用
  * 关键字`arguments` -- 只在函数内部起作用，并且永远指向当前函数的调用者传入的所有参数，类似`Array`
  * _rest_参数
    * `function foo(a, b, ...rest) { ... }`
    * 只能写在最后，前面用`...`标识
    * 若连正常定义参数都没填满，_rest_参数会接收一个空数组（不是`undefined`）
* 变量作用域
  * 变量作用域是函数内部
  * 变量提升
  * 全局作用域 --   默认全局对象`window`
  * 名字空间
    * 全局变量会绑定到`window`
  * 局部作用域
    * `let`代替`var`申明一个块级作用域变量
  * `const`关键字定义常量
* 方法
  * `this`关键字
    * strict 模式下函数的`this`指向`undefined`
    * 函数`apply()`接收两个参数，第一个是需要绑定的`this`变量，第二个是函数本身的参数
    * 与`apply()`类似的方法`call()`
      * `Math.max.apply(null, [3, 5, 4]);` -- `apply()`把参数打包成`Array`再传入
      * `Math.max.call(null, 3, 5, 4);`    -- `call()`把参数按顺序传入
* 装饰器
* 高阶函数
  * JavaScript 的函数其实都指向某个变量。既然变量可以指向函数，函数的参数能接收变量，那么一个函数就可以接收另一个函数作为参数，这种函数就称之为高阶函数。
  * `map` / `reduce` / `filter` / `sort`
* 闭包
* 箭头函数
  * `x => x * x`相当于`function (x) { return x * x; }`
  * `this`，词法作用域
* generator
  * `function* ... { ... yield ... }`
  * `next()`方法
  * `for ... of`循环

## 标准对象

* `Date`
  * 表示日期和时间
  * 时间戳 -- 一个自增的整数，它表示从1970年1月1日零时整的GMT时区开始的那一刻，到现在的毫秒数
  * JavaScript 月份范围 0 -- 11
* `RegExp`
  * 正则表达式
* `JSON`
  * JavaScript Object Notation的缩写，是一种数据交换格式

## 面向对象的编程

* 原型对象 / 基于原型创建一个新对象`Object.create()`
* 创建对象
  * `{ ... }`构造
  * 构造函数，关键字`new`调用
* 原型继承
  1. 定义新的构造函数，并在内部用`call()`调用希望“继承”的构造函数，并绑定`this`
  2. 借助中间函数`F`实现原型链继承，最好通过封装的`inherits`函数完成
  3. 继续在新的构造函数的原型上定义新方法
* `class`继承 -- `extends`关键字

## 浏览器相关

* 浏览器对象
  * `window`对象不但充当全局作用域，而且表示浏览器窗口
    * `window.innerWidth` / `window.innerHeight`
    * `window.outerWidth` / `window.outerHeight`
  * `navigator`对象表示浏览器信息
  * `screen`对象表示屏幕的信息
  * `location`对象表示当前页面的 URL 信息
  * `ducument`对象表示当前页面，是整个 DOM 树的根节点
* 操作 DOM
  * 获取 DOM 节点
    * `document.getElementById()`
    * `document.getElementsByTagName()`
    * `document.getElementsByClassName()`
    * `document.querySelector()`
    * `document.querySelectorAll()`
  * 更新
    * `innerHTML`
    * `innerText`   -- 不返回隐藏元素的文本
    * `textContent` -- 返回所有文本
  * 添加
    * `appendChild`
    * `insertBefore`
  * 删除
    * `removeChild`
  * 遍历
* 操作表单
  * 获取值 -- `value` / `checked`
  * 设置值
  * 提交表单
    * 通过`<form>`元素的`submit()`方法提交一个表单
    * 响应`<form>`本身的`onsubmit`事件
* 操作文件
* AJAX
* Promise
* Canvas
