---
title: JavaScript笔记——事件
date: 2018-04-17 14:58:33
tags: JavaScript
---


# DOM事件流

事件流是指从页面接受事件的顺序。在“DOM2级事件”中规定事件流分为三个阶段：事件捕获阶段、处于目标阶段和时间冒泡阶段。

事件捕获阶段为截获事件提供了机会。
事件冒泡阶段可以对事件做出响应。

一个例子：单击`<div>`元素会按照如图所示顺序触发事件：
![event](/images/javascript3_event.png)


“DOM2级事件”规范明确要求捕获阶段不会涉及事件目标，但是大多数浏览器都会在捕获阶段触发事件对象上的事件。所以就有两次机会在目标对象上操作事件。

# 事件处理程序

为元素添加事件的方法有三种，1，直接在html中的元素的属性上添加；2，通过document的get方法获取到元素，然后为元素的方法设个一个函数；3，通过addEventListener添加

1， HTML事件处理程序

```javascript
//事件的函数直接在双引号中实现
<input type="button" value="click me" onclick="alert('clicked')" />
  
//先实现一个函数
<script>
        function showMessage() {
            alert("a message");
        }
</script>
<input type="button" value="Show Message" onclick="showMessage()">
```

2， DOM0级事件处理程序

```javascript
var btn = document.getElementById("myBtn");
btn.onclick = function() {
    alert("a message");
}
```

如果要删除可以把onclick置为null。

3， DOM2级事件处理程序

“DOM2级事件”定义了两个操作事件的方法，addEventListener()和removeEventListener()。这两个方法都接受三个参数：事件名，函数，bool值。bool值为true表示在捕获阶段调用事件处理程序。false表示在事件冒泡阶段调用事件处理程序。remove的函数必须和add时的函数为同一个，所以如果add时使用的匿名函数就无法移除。

```javascript
 var btn = document.getElementById("myBtn");
btn.addEventListener("click", function() {
    alert("Event Lister Handler")
}, false);
```

如果同时添加了DOM0事件处理程序和DOM2事件处理程序，那执行结果如何呢，看下边的例子

```javascript
var div = document.getElementById("mydiv");
div.onclick = function() {
    console.log("DOM0事件 onclick");
}
div.addEventListener("click", function(){
    console.log("捕获div");
}, true);
 
div.addEventListener("click", function(){
    console.log("冒泡div");
}, false);

 
 
//输出结果
DOM0事件 onclick
捕获div
冒泡div
```

HTML事件处理程序和DOM0事件处理程序是冲突的，后面的会覆盖前面的。

# iOS中的响应链

“DOM2级事件”的规范感觉和iOS里的响应链比较类似。iOS中通过hit-testing找first responder的过程就相当于事件捕捉过程。而事件通过next responder传递就相当于事件冒泡过程。不过需要注意next responder的路径问题。

- UIView对象，如果view是UIViewController对象的root view，则next responder是UIViewController对象，否则是view的superview。
- UIViewController对象，如果UIViewController对象是一个window的rootViewController，则next responder是这个window；如果这个viewcontroller是被另一个viewcontroller present出来的，则next responder是presenting viewcontroller；如果viewcontroller是另一个viewcontroller的childViewController，则next responser是这个viewcontroller的view的superview。
- UIWindow对象，next responder是UIApplication对象
- UIApplication对象，next responder是app delegate，这个app delegate需要是一个UIResponder的实例，而不是view、viewcontroller或者app本身。


备注：JavaScript笔记系列的资料来自《JavaScript高级程序设计》、[ECMAScript 6 入门](http://es6.ruanyifeng.com/)、[TypeScript中文文档](https://www.tslang.cn/docs/home.html)和[CoffeeScript官方文档](http://coffeescript.org/)。

参考：[Understanding Event Handling, Responders, and the Responder Chain](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/understanding_event_handling_responders_and_the_responder_chain)