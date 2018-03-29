---
title: JavaScript笔记——创建对象
date: 2018-03-11 11:52:57
tags: JavaScript
---


# JavaScript

## 工厂模式

``` javascript
function createPerson(name, age, job) {
    var o = new Object();
    o.name = name;
    o.age = age;
    o.job = job;
    o.sayName = function(){
        alert(this.name);
    };
    return o;
}
```

工厂模式虽然解决了创建多个相似对象的问题，但是没有解决对象识别的问题（即不知道对象的类型）。

## 构造函数

``` javascript
 function Person(name, age, job) {
    this.name = name;
    this.age = age;
    this.job = job;
    this.sayName = function() {
        alert(this.name);
    }; //等同于this.sayName = New Function("alert(this.name)");
}
var person1 = new Person("Nicholas", 29, "Software Engineer");
var person2 = new Person("Greg", 25, "Nurse");
```

调用构造函数会经过四个步骤：

1. 创建一个新对象
2. 将构造函数的作用域赋给新对象(因此this就指向了这个新对象)
3. 执行构造函数中的代码（为这个新对象添加属性
4. 返回新对象

通过这种方式创建的对象都含有一个constructor的属性，指向Person。

```javascript

alert(person1 instanceof Object); //true 所有对象都继承自Object对象
alert(person1 instanceof Person); //true
```

构造函数的问题：

1. 每个方法都要在每个实例上创建一遍，即person1和person2的sayName函数不是同一个
2. 构造函数也是函数，这样使用也会在全局作用域创建一个Person的函数

对于问题1的解决方法如下：
```javascript
function Person(name, age, job) {
    this.name = name;
    this.age = age;
    this.job = job;
    this.sayName = sayName;
}
 
function sayName() {
    alert(this.name);
}
```

这样还是存在问题2。

## 原型模式

每个函数都有一个prototype(原型)属性，这个属性是一个指针，指向一个对象(即原型对象)，而这个对象的用于是包含可以由特定类型的所有实例共享的属性和方法。

```javascript
function Person() {
}
 
Person.prototype.name = "Nicholas";
Person.prototype.age = 29;
Person.prototype.job = "Software Engineer";
Person.prototype.sayName = function() {
    alert(this.name);
};
 
var person1 = new Person();
```

### 原型对象

无论什么时候，只要创建了一个新函数，就会更具一组特定的规则为该函数创建一个prototype属性，这个属性执行函数的原型对象。在默认情况下，所有原型对象都会自动获得一个constructor属性，这个属性是一个指向prototype属性所在的函数的指针。

通过这样方式创建的对象，会默认包含一个叫[[Prototype]]的属性，也指向了原型对象，这个对象没有标准的访问方式，再试Firefox、Safari和Chrome上都实现了一个__proto__属性来访问这个。

如图：
![prototype](/images/javascript0_prototype.png)

属性和方法的查找顺序，先在对象内部查找，没有的话再去原型对象中查找。设置对象的属性时，并不会改变原型对象中相应属性的值，这个值会被直接存到对象自身中。即使属性再次设置为null，也不会再去原型对象中查找。不过可以使用delete函数再次恢复指向原型的链接。对象的constructor属性也是一样的。

## 组合使用构造函数模式和原型模式

```javascript
function Person(name, age, job) {
    this.name = name;
    this.age = age;
    this.job = job;
    this.friends = ["Shelby", "Court"];
}
 
Person.prototype = {
    constructor: Person,
    sayName: function() {
        alert(this.name);
    }
}
 
var person1 = new Person("Nicholas", 29, "Software Engineer");
```

组合模式其实是属性放在对象中，而方法放在原型对象中。

# ES6

ES6提供了接近传统语言的写法，引入Class（类）的概念，通过class关键字定义类。

```javascript
class Point {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }
    toString() {
        return '(' + this.x + ', ' + this.y + ')';
    }
}
  
var point = new Point(3, 4);
```

class的写法是一个语法糖，让对象原型的写法更加清晰，更像面向对象编程的语法。class里的方法定义不需要添加function关键字。方法与方法之间不需要逗号或者分号来分割。

## constructor

构造方法是类的默认方法，一个类必须有constructor方法，如果没有就会默认添加一个空的构造方法。构造方法默认返回实例对象。

类的初始化必须使用new，否则会报错。

## Class表达式

与函数一样，类也可以使用表达式的形式定义。

```javascript
const MyClass = class Me {
    getClassName() {
        return Me.name;
    }
};
let inst = new MyClass();
inst.getClassName(); //Me
Me.name // ReferenceError: Me is not defined
```

# TypeScript

可以声明属性，并指定类型。属性声明需要使用分号分隔。

```typescript
class Greeter {
    greeting: string;
    constructor(message: string) {
        this.greeting = message;
    }
    greet() {
        return "Hello, " + this.greeting;
    }
}
 
let greeter = new Greeter("world");
```

# CoffeeScript

```coffeescript
class Animal
  constructor: (@name) ->
 
  move: (meters) ->
    alert @name + " moved #{meters}m."
  
  
//转换之后的JS代码
var Animal;
Animal = class Animal {
  constructor(name) {
    this.name = name;
  }
 
  move(meters) {
    return alert(this.name + ` moved ${meters}m.`);
  }
 
};
```

写法和ruby类似，以缩进为依据判断代码块。


备注：JavaScript笔记系列的资料来自《JavaScript高级程序设计》、[ECMAScript 6 入门](http://es6.ruanyifeng.com/)、[TypeScript中文文档](https://www.tslang.cn/docs/home.html)和[CoffeeScript官方文档](http://coffeescript.org/)。