---
title: 对象创建&类的实现
date: 2019-03-12 16:26:22
tags:
---

ES5的标准里没有类的概念, 所以大牛们创造了很多创建对象和模拟类的方法

## 工厂模式
工厂模式是一种常用的设计模式,将对象的创建过程抽象到一个函数中
```javascript
function createStudent(name, age, id) {
  var o = new Object(); 
  o.name = name;
  o.age = age;
  o.id = id;
  o.getId = function() {
    return o.id;
  }
  return o;
}

var obj = createStudent('Shadow', 18, 13201);
```
## 构造函数模式
构造函数是用来创建某种特定类型对象的函数,比如原生的Object, Array等,
当然我们也可以自己创建构造函数:
```javascript
function Student(name, age, id) {
  this.name = name;
  this.age = age;
  this.id = id;
  this.getId() = function() {
    return this.id;
  }
}

var obj = new Student('Shadow', 18, 13201);
```
注意此构造函数与工厂函数的区别:
1. 命名区别: createStudent --> Student, 此处构造函数命名与类的命名一致;
2. 未在函数内部创建新对象, 而是直接把属性和方法赋值给了this;
3. 未写return语句;
4. 创建对象时使用了new操作符.
这些区别的原因就在new操作符的作用上(引自《JavaScript高级程序设计(第3版)》):
> （1）创建一个新对象；
> （2）将构造函数的作用域赋给新对象（因此this就指向了这个新对象）；
> （3）执行构造函数中的代码（为这个新对象添加属性）；
> （4）返回新对象。

💡tips: 关于return语句, 如果构造函数内写了return语句, 那么会覆盖new最后一步返回return语句值(必须是对象,⚠️ null不是对象!!):
```javascript
function Student(name, age, id) {
  this.name = name;
  this.age = age;
  this.id = id;
  this.getId = function() {
    return this.id;
  }
  return { test: 1 };
}

var obj = new Student('Shadow', 18, 13201);
// obj: {test: 1}
```
构造函数存在的问题:
```javascript
function Student(name, age, id) {
  this.name = name;
  this.age = age;
  this.id = id;
  this.getId = function() {
    return this.id;
  }
}

var shadow = new Student('Shadow', 18, 13201);
var shine = new Student('Sine', 16, 13201);
```
如上所见, 我们定义了一个Student类, 包含一个getId的方法, 之后用这个类创建了两个对象
'shadow'和'shine', 它们的getId方法都是不同的引用, 而实际上这样是没有必要的, 因为getId的作用一致,
所以不同的对象使用同一个方法就可以了, 这样造成了空间的浪费

## 组合模式(构造函数+原型模式)
在上一篇原型和继承的文章中我们介绍了原型, 原型可以用来存放共享的属性和方法, 构造函数用来新建实例自定义的属性
```javascript
function Student(name, age, id) {
  this.name = name;
  this.age = age;
  this.id = id;
}
Student.prototype.getId = function() {
  return this.id;
}

var shadow = new Student('Shadow', 18, 13201);
```
我们一般采取这种方式作为ES5实现类的默认方式

## 寄生构造函数模式
这种模式的作用仅仅是封装创建对象的代码, 返回新创建的对象, 但是使用方法跟构造函数一样.
```javascript
function MyArray(arr) {
  var o = new Array(arr);

  o.toMyString = function() {
    return this.join(',');
  }

  // 重写构造函数的返回
  return o;
}

var arr = new MyArray([1, 2, 3]);
```
我们可以看到这个方法通过return重写了new时的返回来达到效果,
假设我们想创建一个具有额外方法的特殊数组。由于不能直接修改Array构造函数，因此可以使用这个模式

## 稳妥构造函数模式
构建除了使用暴露出来的方法,否则外部无法访问的对象
```javascript
function Student(name) {
  var o = new Object();
  o.getName = function() {
    return name;
  }
  return o;
}

var shadow = new Student('Shadow');
```
上述模式创建的对象, 除了使用getName方法之外, 没有其他途径得到shadow的name,
实现原理是通过闭包
