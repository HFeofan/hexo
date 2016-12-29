---
title: 运行时——消息转发
date: 2016-12-29 22:45:48
tags:
---

如果调用一个没有实现的方法，会触发一个`NSInvalidArgumentException`的异常，虽然看到这种异常，通常做法是将这个方法实现。但是还是需要理解背后的转发机制，对于写底层库也是有帮助的，比如隐藏实现细节或者实现*多继承*。

# 消息转发
使用`instrumentObjcMessageSends(YES);`可以将消息调用存到`/private/tmp/msgSends-xxx`文件里。测试代码如下
``` objectivec
(void)instrumentObjcMessageSends(YES);
NSObject *obj = [NSObject new];
[obj performSelector:@selector(testMethod)];
```

输出
```
+ NSObject NSObject new
- NSObject NSObject init
- NSObject NSObject performSelector:
+ NSObject NSObject resolveInstanceMethod:
+ NSObject NSObject resolveInstanceMethod:
- NSObject NSObject forwardingTargetForSelector:
- NSObject NSObject forwardingTargetForSelector:
- NSObject NSObject methodSignatureForSelector:
- NSObject NSObject methodSignatureForSelector:
- NSObject NSObject class
- NSObject NSObject doesNotRecognizeSelector:
- NSObject NSObject doesNotRecognizeSelector:
```

然后实现一个自定义的类，重载上边的方法，打断点看调用栈。

- `resolveInstanceMethod:`[图1]

![breakpoint1](/images/runtime1-break1.png)

第一次调用，所以没有cache，开始查找或者转发。然后调用`resolveInstanceMethod`方法尝试解决。

- `forwardingTargetForSelector:`[图2]

![breakpoint2](/images/runtime1-break2.png)

`resolveInstanceMethod`没有解决，进入转发，调用`forwardingTargetForSelector`尝试转发给其他对象。这里如果返回`self`的话就会进入无限循环。

- `methodSignatureForSelector:`

断点在这里的调用栈和图2一样，只不多最顶层是`methodSignatureForSelector`。看[Rumtime Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html)介绍说，在抛出异常之前会调用`forwardInvocation:`这个方法来指示接下来的去处。

所以，调用这个方法的目的应该是使用一个`methodSignature`来初始化`NSInvocation`，将消息的参数也保存到`NSInvocation`中，把这个`NSInvocation`传给`forwardInvocation:`。`NSObject`的`forwardInvocation:`会触发`doesNotRecognizeSelector:`方法。

如果`methodSignatureForSelector`返回nil的话，就会触发`doesNotRecognizeSelector:`。

- `doesNotRecognizeSelector:`

最后一步，在NSObjct中这个方法抛出一个`NSInvalidArgumentException`异常。

# 其它细节

- `respondsToSelector:`和`instancesRespondToSelector:`

使用这两个方法判断时，如果没有该方法，会触发`resolveInstanceMethod:`方法，可以在这个方法动态地添加IMP。

- 转发与继承

虽然可以用转发模仿[多]继承，但是像`NSObject`的`respondsToSelector:`和`isKindOfClass:`这类方法只关注继承结构，并不在转发链上。`resolveInstanceMethod:`也不在转发链上，从调用栈上可以看出。所以如果需要尽可能的模拟多继承，需要依照转发逻辑重载这些方法。

# 参考
1. https://segmentfault.com/a/1190000004202319