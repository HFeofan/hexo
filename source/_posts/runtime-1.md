---
title: 运行时前篇——对象模型
date: 2016-12-06 10:22:38
tags: iOS
---

在项目中通常使用一些运行的方法做一些hack，或者在开发一个库时使用运行时解耦。这些操作涉及到Objective-C的对象模型，本篇内容会讨论OC的对象模型。

这里需要借助`clang -rewrite-objc`这个命令，它可以把OC代码转成C/C++代码。

示例代码：
``` objective-c
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface XYTree : NSObject
@property (nonatomic, strong) NSString *xyName;
- (void)doTask;
- (void)doOtherTask;
@end

@interface XYTree ()
@property (nonatomic, strong) NSString *exName;
@end

@implementation XYTree
- (void)doTask
{ 
}
@end

@interface XYTree (XYTreeCategory)
@property (nonatomic, strong) NSString *categoryName;
@end

@implementation XYTree (XYTreeCategory)
- (void)setCategoryName:(NSString *)categoryName
{
    objc_setAssociatedObject(self, @selector(categoryName), categoryName, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
- (NSString *)categoryName
{
    return objc_getAssociatedObject(self, _cmd);
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        XYTree *object = [XYTree new];
        object.xyName = @"xyname";
        object.exName = @"exname";
        object.categoryName = @"categoryName";
        NSLog(@"%@, %@, %@", object.xyName, object.exName, object.categoryName);
    }
    return 0;
}
```
在终端执行`clang -rewrite-objc main.m`后，在当前目录下生成一个main.cpp的文件。

