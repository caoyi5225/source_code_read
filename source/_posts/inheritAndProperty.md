---
title: 原型与继承
date: 2019-03-04 17:25:39
tags:
---

原型和继承的问题是JS的重要基础, 也是面试中经常闻到的老生常谈,
今天就来看一下原型和继承的相关知识

### 原型
概念(引自《Javascript权威指南》第六版):
> 每一个javascript对象(null除外)都和另一个对象相关联.“另一个”对象就是我们熟知的原型, 每一个对象都从原型继承属性.

注意这里的概念是“原型”, 接下来还有另外一个概念叫“原型对象”, 它们是不同的!

### 原型对象
概念(引自《Javascript权威指南》第六版):
> “每一个函数都包含一个prototype属性，这个属性是指向一个对象的引用，这个对象称做“原型对象”（prototype object）。”

比如: 
```javascript
Object.prototype
Array.prototype
Date.prototype
```
“原型对象”是创建新对象是原型要指向的对象
注: 函数的constructor是Function, 原型是Function.prototype

### new 关键字
摘自[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/new)
> 当代码 var obj = new Foo(...) 执行时，会发生以下事情：
> 1. 一个继承自 Foo.prototype 的新对象被创建
> 2. 使用指定的参数调用构造函数 Foo ，并将 this 绑定到新创建的对象。new Foo 等同于 new Foo()，也就是没有指定参数列表，Foo 不带任何参数调用的情况
> 3. 由构造函数返回的对象就是 new 表达式的结果。如果构造函数没有显式返回一个对象，则使用步骤1创建的对象。（一般情况下，构造函数不返回值，但是用户可以选择主动返回对象，来覆盖正常的对象创建步骤）

### 原型链
js在一个对象内寻找属性时,会先在内部查找,如果找不到就会到原型上查找,再找不到,就到原型的原型上找,直到null:
var obj = {a: 1}
obj.b => obj.\_\_proto\_\_ => obj.\_\_proto\_\_.\_\_proto\_\_ => ····· => Object.prototype => null
