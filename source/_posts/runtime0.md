---
title: 运行时——NSInvocation与消息机制
date: 2016-12-21 15:28:19
tags: iOS
---

以前在学习iOS开发时，接触到`NSInvocation`这个类，觉得这个类很强大，但是复杂的设置过程也很让人迷惑（当时不怎么理解）。而且在平时的开发中基本上没有用到过，从此消失在视野里了。在回顾runtime的知识时，发现`NSInvocation`和消息机制有些相像。所以再次认识一下`NSInvocation`。

# NSInvocation

## 用法

``` ObjectiveC
NSMethodSignature *methodSignature = [self methodSignatureForSelector:@selector(doWork:)];
NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
invocation.selector = @selector(doWork:);
NSInteger value = 2;
[invocation setArgument:&value atIndex:2];
[invocation invokeWithTarget:self];
```

需要注意的是，已经生成的`NSInvocation`对象的`methodSignature`不能更改，并且创建的时候使用`invocationWithMethodSignature`类方法创建。不能通过`alloc`和`init`的方式创建。生成的对象并不会retain设置的参数，需要手动通过`argumentsRetained`方法retain。

<!--more-->

## 使用场景
- 将`NSInvocation`对象保存起来在将来使用。
用《Head First设计模式》中的命令模式做个例子，实现一个只有开和关按钮的遥控器。

``` ObjectiveC
//灯
@interface Light : NSObject
- (void)open;
- (void)close;
@end

@interface Light (Command)
- (NSInvocation *)onCommand;
- (NSInvocation *)offCommand;
@end

@implementation Light
- (void)open
{
    NSLog(@"Light opened");
}
- (void)close
{
    NSLog(@"Light closed");
}
@end

@implementation Light (Command)
- (NSInvocation *)onCommand
{
    NSMethodSignature *methodSignature = [self methodSignatureForSelector:@selector(open)];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
    invocation.target = self;
    invocation.selector = @selector(open);
    [invocation retainArguments];
    return invocation;
}
- (NSInvocation *)offCommand
{
    NSMethodSignature *methodSignature = [self methodSignatureForSelector:@selector(close)];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
    invocation.target = self;
    invocation.selector = @selector(close);
    [invocation retainArguments];
    return invocation;
}
@end

//遥控器
@interface RemoteControl : NSObject
@property (nonatomic, strong) NSInvocation *onSlot;
@property (nonatomic, strong) NSInvocation *offSlot;
- (void)onButtonWasPressed;
- (void)offButtonWasPressed;
@end
@implementation RemoteControl

- (void)onButtonWasPressed {
    [self.onSlot invoke];
}
- (void)offButtonWasPressed
{
    [self.offSlot invoke];
}
@end

//测试
Light *light = [Light new];
RemoteControl *remoteControl = [RemoteControl new];
remoteControl.onSlot = light.onCommand;
remoteControl.offSlot = light.offCommand;
[remoteControl onButtonWasPressed];
```

- 和Timer一起使用

``` C
[NSTimer scheduledTimerWithTimeInterval:1 invocation:invocation repeats:NO];
```

感觉这种场景很少用到。需要注意的是，timer会让invocation去retain它的参数。


# 方法
看到上边的例子提到了`NSMethodSignature`和经常使用的`@selector`函数。所以接下来看看它们表示的内容和作用是什么。

## NSMethodSignature
方法签名的初始化使用的是一个返回值和参数经过编码的数组，编码使用`@encode()`编译指令。`-(void)doWork:(NSInteger)value`的方法签名输出如下：

```
    number of arguments = 3
    frame size = 224
    is special struct return? NO
    return value: -------- -------- -------- --------
        type encoding (v) 'v'
        flags {}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 0, size adjust = 0}
        memory {offset = 0, size = 0}
    argument 0: -------- -------- -------- --------
        type encoding (@) '@'
        flags {isObject}
        modifiers {}
        frame {offset = 0, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 1: -------- -------- -------- --------
        type encoding (:) ':'
        flags {}
        modifiers {}
        frame {offset = 8, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
    argument 2: -------- -------- -------- --------
        type encoding (q) 'q'
        flags {isSigned}
        modifiers {}
        frame {offset = 16, offset adjust = 0, size = 8, size adjust = 0}
        memory {offset = 0, size = 8}
```

可以看到返回值的编码是`v`，参数有三个，分别是`self`、`_cmd`、`value`。`self`的编码是`@`，`_cmd`的编码是`:`，`NSInteger`类型的value编码是`q`。

关于类型编码可以看[Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

## SEL
通过`@selector()`编译指令可以得到一个类型为`SEL`的变量，那么`SEL`是什么样的结构呢？在`objc.h`中找到如下定义：

``` C
typedef struct objc_selector *SEL;
```

那么`struct objc_selector`是什么呢？在stackoverflow上查到，这个定义根据平台决定，在NeXT Objective-C Runtime中是一个C字符串。所以可以通过`NSLog(@"SEL = %s", @selector(blah));`打印。参见[stackoverflow](http://stackoverflow.com/questions/28581489/what-is-the-objc-selector-implementation)

## IMP
同样地，在`objc.h`中也看到了`IMP`的定义

``` C
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```

`IMP`指向一个方法的函数实现。

## Method
根据苹果的开源代码[objc](https://opensource.apple.com/tarballs/objc4/)找到如下定义：

``` C
//objc-private.h
typedef struct method_t *Method;

//objc-runtime-new.h
struct method_t {
    SEL name;
    const char *types;
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

这个和上篇文章中的`struct _objc_method`超不多。可以看到`Method`里包含了`SEL`和`IMP`，`types`字段是方法的编码，可以和上片文章里生成的方法对比一下`{(struct objc_selector *)"doTask", "v16@0:8", (void *)_I_XYTree_doTask`。结合上边的编码说明，可以了解这个`types`的内容。

# 消息机制
在代码中的方法调用，都会转换成使用`objc_msgSend`的调用。如`[receiver message]`会转换成`objc_msgSend(receiver, selector)`。如果有参数的话，形式就是`objc_msgSend(receiver, selecotr, arg1, arg2, ...)`。

`objc_msgSend`会通过selector找到receiver上的方法实现`IMP`，然后将参数传给`IMP`。方法查找的过程如下图[图片来源](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Art/messaging1.gif)
![messaging1](/images/runtime0_messaging1.gif)

给对象发送消息时，消息机制会通过`isa`查找方法。如果没有找到，会继续找`superclass`里的方法，直到`NSOject`。为了提高查找的效率，rumtime会缓存使用过的方法。

对于普通的对象方法调用，会转换成`objc_msgSend`，使用`super`的方法调用会转换成`objc_msgSendSuper`，第一个参数是`struct objc_super *`的super指针。如果返回值是结构体的话会转成`objc_msgSend_stret`或者`objc_msgSendSuper_stret`。

``` C
// message.h
/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
```

当然也可以使用IMP直接调用方法，不过不建议这种使用：
``` C
void(*click)(id, SEL);
click = (void(*)(id, SEL))[self methodForSelector:@selector(buttonDidClicked)];
click(self, @selector(buttonDidClicked));
```

# Swizzle

在项目里经常使用Swizzle交换方法。常规的一种做法如下：
``` C
Class class = [self class];
SEL originalSelector = @selector(viewWillAppear:);
SEL swizzledSelector = @selector(xxx_viewWillAppear:);

Method originalMethod = class_getInstanceMethod(class, originalSelector);
Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

BOOL didAddMethod =
    class_addMethod(class,
        originalSelector,
        method_getImplementation(swizzledMethod),
        method_getTypeEncoding(swizzledMethod));

if (didAddMethod) {
    class_replaceMethod(class,
        swizzledSelector,
        method_getImplementation(originalMethod),
        method_getTypeEncoding(originalMethod));
} else {
    method_exchangeImplementations(originalMethod, swizzledMethod);
}
```
其实还可以判断下`originalMethod`和`swizzledMethod`是否有值，没有的直接返回。
- 添加成功的情况分析

| ① class_addMethod| ② class_replaceMethod|
| ------------- |:-------------:|
| originalSelector 对应 swizzledIMP  | originalSelector 对应 swizzledIMP     |
| swizzledSelector 对应 swizzledIMP  | swizzledSelector 对应 super.originalIMP     |

- 添加不成功的情况分析

| ① class_addMethod| ② method_exchangeImplementations|
| ------------- |:-------------:|
| originalSelector 对应 originalIMP  | originalSelector 对应 swizzledIMP     |
| swizzledSelector 对应 swizzledIMP  | swizzledSelector 对应 originalIMP     |

如果我们想用一个block和一个方法交换的话可以这样做：
``` C
IMP class_swizzleSelector(Class clazz, SEL selector, id newImplementationBlock)
{
    IMP newImplementation = imp_implementationWithBlock(newImplementationBlock);
    Method method = class_getInstanceMethod(clazz, selector);
    if (! method) {
        return NULL;
    }

    const char *types = method_getTypeEncoding(method);
    class_addMethod(clazz, selector, imp_implementationWithBlock(^(__unsafe_unretained id self, va_list argp) {
        struct objc_super super = {
            .receiver = self,
            .super_class = class_getSuperclass(clazz)
        };
        
        id (*objc_msgSendSuper_typed)(struct objc_super *, SEL, va_list) = (void *)&objc_msgSendSuper;
        return objc_msgSendSuper_typed(&super, selector, argp);
    }), types);
    
    return class_replaceMethod(clazz, selector, newImplementation, types);
}
```

通过swizzledSelector去交换方法需要注意不要和其他库或者其他人写的代码方法名重复，要不然会出现期望外的结构。而使用swizzledIMP交换的话没有这中问题。


# 参考

1. http://nshipster.cn/method-swizzling/
