---
title: JavaScript笔记——继承
date: 2018-03-22 19:09:34
tags: JavaScript
---


# JavaScript

将子类型的prototype设置为超类型的实例，就实现了一个继承关系，代码如下：

```javascript
function Person(name) {
    this.name = name;
}
Person.prototype.sayName = function() {
    alert(this.name);
};
 
function Student(className) {
    this.className = className;
}
Student.prototype = new Person();
Student.prototype.constructor = Student; //手动设置constructor
Student.prototype.sayClassName = function() {
    alert(this.className);
};
 
var instance = new Student();
instance.name = "Alex";
instance.className = "3年级2班";
instance.sayName();
instance.sayClassName();
```

结合《创建对象》中的原型链，以及所有对象都是继承自Object，得到完整关系的原型链。

![objcect](/images/javascript1_prototype.png)

每个原型对象有个constructor的指针指向构造函数，实例中的Student Prototype没有画出，因为需要手动设置，设置之后就会和Person Prototype一样有完整的关系链。

通过这种方式继承会有一个问题，就是子类型中的属性会共享，如果Person中有个colors属性，其中一个子类型的实例push了一个元素进去，其他实例也会被添加了一个元素，如果解决这种问题，需要使用“借用构造函数”的写法，如果熟悉其他编程语言的话，会对这种写法不会感到陌生，就是在子类型的构造函数中调用父类的构造函数。写法如下：

```javascript
function Student(className) {
    Person.call(this)
    this.className = className;
}
```

# ES6
使用extends关键字实现继承，如：

```javascript

class Point {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }
 
    toString() {
        return this.x + ", " + this.y;
    }
}
 
class ColorPoint extends Point {
    constructor(x, y, color) {
        super(x, y);
        this.color = color;
    }
 
    toString() {
        return this.color + " " + super.toString();
    }
}
```

## 类的构造过程

与ES5不同，ES6是先创造父类的实例，然后用子类的构造函数修改该对象。所有如果在子类的constructor方法中没有调用super就对this的属性赋值会造成错误。

## Super

在子类的constructor方法中super表示的是父类的构造函数，而不是父类的原型对象。在非constructor方法中指的是父类的原型对象。

# TypeScript

TypeScript的继承写法和ES6一样，多了一些访问控制修饰符，关于访问控制修饰符以后再详细说明。

# CoffeeScript

在CoffeeScript中也是使用extends关键字实现继承，如下：

```coffeescript
class Animal
  constructor: (@name) ->
 
  move: (meters) ->
    alert @name + " moved #{meters}m."
 
 
class Snake extends Animal
  move: ->
    alert "Slithering..."
    super 5
```

这里不一样地方就是函数的重载，参数与父类中参数个数不一致也会当做函数重载，而且super的用法和ES6也不一样，可以直接用super当做父类中方法，和ruby调用父类方法一样。


备注：JavaScript笔记系列的资料来自《JavaScript高级程序设计》、[ECMAScript 6 入门](http://es6.ruanyifeng.com/)、[TypeScript中文文档](https://www.tslang.cn/docs/home.html)和[CoffeeScript官方文档](http://coffeescript.org/)。