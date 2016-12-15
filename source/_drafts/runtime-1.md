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
        XYTree *object = [XYTree new];
        object.xyName = @"xyname";
        object.exName = @"exname";
        object.categoryName = @"categoryName";
        NSLog(@"%@, %@, %@", object.xyName, object.exName, object.categoryName);
    }
    return 0;
}
```
在终端执行`clang -rewrite-objc main.m`后，在当前目录下生成一个main.cpp的文件。通过解析生成的main.cpp可以来窥探OC的内部实现。

# 初始化
``` c
static void OBJC_CLASS_SETUP_$_XYTree(void ) {
    OBJC_METACLASS_$_XYTree.isa = &OBJC_METACLASS_$_NSObject;
    OBJC_METACLASS_$_XYTree.superclass = &OBJC_METACLASS_$_NSObject;
    OBJC_METACLASS_$_XYTree.cache = &_objc_empty_cache;
    OBJC_CLASS_$_XYTree.isa = &OBJC_METACLASS_$_XYTree;
    OBJC_CLASS_$_XYTree.superclass = &OBJC_CLASS_$_NSObject;
    OBJC_CLASS_$_XYTree.cache = &_objc_empty_cache;
}
#pragma section(".objc_inithooks$B", long, read, write)
__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CLASS_SETUP[] = {
    (void *)&OBJC_CLASS_SETUP_$_XYTree,
};
```
# 方法列表
```
static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[5];
} _OBJC_$_INSTANCE_METHODS_XYTree __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    5,
    {{(struct objc_selector *)"doTask", "v16@0:8", (void *)_I_XYTree_doTask},
    {(struct objc_selector *)"xyName", "@16@0:8", (void *)_I_XYTree_xyName},
    {(struct objc_selector *)"setXyName:", "v24@0:8@16", (void *)_I_XYTree_setXyName_},
    {(struct objc_selector *)"exName", "@16@0:8", (void *)_I_XYTree_exName},
    {(struct objc_selector *)"setExName:", "v24@0:8@16", (void *)_I_XYTree_setExName_}}
};
```

# 属性列表
```
static struct /*_prop_list_t*/ {
    unsigned int entsize;  // sizeof(struct _prop_t)
    unsigned int count_of_properties;
    struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_XYTree __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_prop_t),
    1,
    {{"xyName","T@\"NSString\",&,N,V_xyName"}}
};
```

# 成员变量列表
```
static struct /*_ivar_list_t*/ {
    unsigned int entsize;  // sizeof(struct _prop_t)
    unsigned int count;
    struct _ivar_t ivar_list[2];
} _OBJC_$_INSTANCE_VARIABLES_XYTree __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_ivar_t),
    2,
    {{(unsigned long int *)&OBJC_IVAR_$_XYTree$_xyName, "_xyName", "@\"NSString\"", 3, 8},
     {(unsigned long int *)&OBJC_IVAR_$_XYTree$_exName, "_exName", "@\"NSString\"", 3, 8}}
};
```
