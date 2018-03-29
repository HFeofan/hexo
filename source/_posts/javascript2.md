---
title: JavaScript笔记——函数
date: 2018-03-29 19:47:35
tags: JavaScript
---

# 创建函数

```javascript
//方式一
function hello(name) {
    console.log("hello " + name);
}
 
//方式二
var helloFunc = function(name) {
    console.log("hello " + name);
}
 
hello("world");
helloFunc("function");
```

两者的区别：
1. 方式一会有作用域提升，也就是说，可以在生命之前调用，而方式二不能这么做。
2. 方式二可以在条件语句中使用

# 递归

```javascript
function factorial(num) {
    if (num <= 1) {
        return 1;
    } else {
        return num * factorial(num - 1);
    }
}
```
这种写法的函数不能实现把`factorial`赋给普通变量，如`var anotherFactorial = factorial`; 如果这时候把`factorial=null`; 调用anotherFactorial会报错，找不到`factorial`。在`function`的内部，可以访问`arguments.callee`指针，这个指针指向了调用者本身。如下

```javascript
function factorial(num) {
    if (num <= 1) {
        return 1;
    } else {
        return num * arguments.callee(num - 1);
    }
}
```

但是严格模式下，访问`arguments.callee`会报错。可以改成函数表达式的形式，如下

```javascript
var factorial = (function f(num) {
   if (num <= 1) {
       return 1;
   } else {
       return num * f(num - 1);
   }
});
```

# 模仿块级作用域

在JavaScript中没有块级作用域的概念，所以在块语句中定义的变量，实际上是包含在函数中的，如下

```javascript
function outputNumbers(count) {
    for (var i = 0; i < count; i++) {
        alert(i);
    }
    var i;
    alert(i);//这里输出的是count-1的值
}
```

为了解决这个问题，使用匿名函数创建块级作用域，块级作用域的创建方式如下

```javascript
(function(){
    //这里是块级作用域
})();
```

需要使用`()`将`function`声明的匿名函数包起来，要不`function`会被当成函数声明开始，后边的括号会导致错误。使用块级作用域的`outputNumbers`函数。

```javascript
function outputNumbers(count) {
    (function(){
        for (var i = 0; i < count; i++) {
            alert(i);
        }
    })();
    alert(i); // 报错
}
```

# 私有变量

特权方法：把有权访问私有变量和私有函数的共有方法称为特权方法，在构造函数中定义特权方法的模式如下

```javascript
function Person(name) {
    this.getName = function() {
        return name;
    }
    this.setName = function(value) {
        name = value;
    }
}
 
var person = new Person("Nicholas");
console.log(person.getName());
person.setName("Greg");
console.log(person.getName());
```

利用特权方法可以隐藏那些不应该直接修改的数据。在这个例子中，name是Person的属性，并不是所有实例共享的。

## 静态私有变量

声明静态私有变量的模式如下
```javascript
var Person;
(function(){
    var name = "";
     
    Person = function(value) {
        name = value;
    };
 
    Person.prototype.getName = function() {
        return name;
    };
 
    Person.prototype.setName = function(value) {
        name = value;
    };
})();
 
var person1 = new Person("Nicholas");
console.log(person1.getName());
person1.setName("Greg");
console.log(person1.getName());
 
var person2 = new Person("Michael");
console.log(person1.getName());
console.log(person2.getName());
 
 
//输出结果
Nicholas
Greg
Michael
Michael
```

需要注意的是，创建构造函数的时候使用了函数表达式，而不是函数声明。

## 模块模式

前面的模式是为自定义类型创建私有变量和特权方法，模块模式是为单例创建私有变量和特权方法。JavaScript是以字面量的方式创建单例对象的。

```javascript
 //单例模式
var singleton = {
    name: "value",
    method: function() {
        //这里是方法的代码
    }
};
 
//模块模式增强的单例模式
var singleton = function(){
    var privateVariable = 10;
    function privateFunction() {
        return false;
    }
 
    return {
        publicProperty: true,
        publicMethod: function() {
            privateVariable++;
            return privateFunction();
        }
    }
}();
```

注意这里的`function(){}`并没有被`()`括起来，与模拟块级作用域相比，我觉得可能是因为这里是赋值语句，没有歧义，当然也可以用括号括起来。


备注：JavaScript笔记系列的资料来自《JavaScript高级程序设计》、[ECMAScript 6 入门](http://es6.ruanyifeng.com/)、[TypeScript中文文档](https://www.tslang.cn/docs/home.html)和[CoffeeScript官方文档](http://coffeescript.org/)。

