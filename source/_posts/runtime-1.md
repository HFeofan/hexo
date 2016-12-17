---
title: 运行时前篇——对象初始化设置分析
date: 2016-12-06 10:22:38
tags: iOS
---

在项目中通常使用一些运行的方法做一些hack，或者在开发一个库时使用运行时解耦。这些操作涉及到Objective-C的对象模型，本篇内容会讨论OC的对象模型。附ojbc的源码：https://opensource.apple.com/tarballs/objc4/

这里需要借助`clang -rewrite-objc`这个命令，它可以把OC代码转成C/C++代码。

示例代码：
``` Objective-C
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

<!--more-->

打开main.cpp之后发现了一些基本结构，先来看看这些基本结构
# 基本结构体
## struct _class_t
``` C
struct _class_t {
    struct _class_t *isa; 
    struct _class_t *superclass;
    void *cache;
    void *vtable;
    struct _class_ro_t *ro;
};
```

## struct _class_ro_t
``` C
struct _class_ro_t {
    unsigned int flags;
    unsigned int instanceStart;
    unsigned int instanceSize;
    unsigned int reserved;
    const unsigned char *ivarLayout;
    const char *name;
    const struct _method_list_t *baseMethods;
    const struct _objc_protocol_list *baseProtocols;
    const struct _ivar_list_t *ivars;
    const unsigned char *weakIvarLayout;
    const struct _prop_list_t *properties;
};
```

## struct _objc_method
``` C
struct _objc_method {
    struct objc_selector * _cmd;
    const char *method_type;
    void  *_imp;
};
```

## struct _protocol_t
``` C
struct _protocol_t {
    void * isa;  // NULL
    const char *protocol_name;
    const struct _protocol_list_t * protocol_list; // super protocols
    const struct method_list_t *instance_methods;
    const struct method_list_t *class_methods;
    const struct method_list_t *optionalInstanceMethods;
    const struct method_list_t *optionalClassMethods;
    const struct _prop_list_t * properties;
    const unsigned int size;  // sizeof(struct _protocol_t)
    const unsigned int flags;  // = 0
    const char ** extendedMethodTypes;
};
```

## struct _ivar_t
``` C
struct _ivar_t {
    unsigned long int *offset;  // pointer to ivar offset location
    const char *name;
    const char *type;
    unsigned int alignment;
    unsigned int  size;
};
```

## struct _prop_t
``` C
struct _prop_t {
    const char *name;
    const char *attributes;
};
```

## struct _category_t
``` C
struct _category_t {
    const char *name;
    struct _class_t *cls; 
    const struct _method_list_t *instance_methods;
    const struct _method_list_t *class_methods;
    const struct _protocol_list_t *protocols;
    const struct _prop_list_t *properties;
};
```
对基本结构有了认识之后再来看看`XYTree`的初始化设置

# XYTree的初始化
``` C
XYTree *object = ((XYTree *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("XYTree"), sel_registerName("new"));
```
浏览生成的main.cpp，发现了两个inithook
``` C
__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CLASS_SETUP[] = {
    (void *)&OBJC_CLASS_SETUP_$_XYTree,
};

__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CATEGORY_SETUP[] = {
    (void *)&OBJC_CATEGORY_SETUP_$_XYTree_$_XYTreeCategory,
};
```
一个是class的setup，一个是category的setup。

## OBJC_CLASS_SETUP_$_XYTree
``` C
static void OBJC_CLASS_SETUP_$_XYTree(void ) {
    OBJC_METACLASS_$_XYTree.isa = &OBJC_METACLASS_$_NSObject;
    OBJC_METACLASS_$_XYTree.superclass = &OBJC_METACLASS_$_NSObject;
    OBJC_METACLASS_$_XYTree.cache = &_objc_empty_cache;
    OBJC_CLASS_$_XYTree.isa = &OBJC_METACLASS_$_XYTree;
    OBJC_CLASS_$_XYTree.superclass = &OBJC_CLASS_$_NSObject;
    OBJC_CLASS_$_XYTree.cache = &_objc_empty_cache;
}
```
如上，设置了XYTree的metaclass和本身的`isa`、`superclass`、`cache`。

### OBJC_METACLASS_$_XYTree
``` C
extern "C" __declspec(dllexport) struct _class_t OBJC_METACLASS_$_XYTree __attribute__ ((used, section ("__DATA,__objc_data"))) = {
    0, // &OBJC_METACLASS_$_NSObject,
    0, // &OBJC_METACLASS_$_NSObject,
    0, // (void *)&_objc_empty_cache,
    0, // unused, was (void *)&_objc_empty_vtable,
    &_OBJC_METACLASS_RO_$_XYTree,
};
```

### OBJC_CLASS_$_XYTree
``` C
extern "C" __declspec(dllexport) struct _class_t OBJC_CLASS_$_XYTree __attribute__ ((used, section ("__DATA,__objc_data"))) = {
    0, // &OBJC_METACLASS_$_XYTree,
    0, // &OBJC_CLASS_$_NSObject,
    0, // (void *)&_objc_empty_cache,
    0, // unused, was (void *)&_objc_empty_vtable,
    &_OBJC_CLASS_RO_$_XYTree,
};
```

### isa与superClass的指向链
从上边的代码可以看出`XYTree`以及它的`MetaClass`中`isa`和`superClass`的指向关系。下图[[图片来源]](http://www.sealiesoftware.com/blog/class%20diagram.pdf)可以更清楚的描述这点：
![Class Diagram](/images/runtime-1_classdiagram.png)



## OBJC_CATEGORY_SETUP_$_XYTree_$_XYTreeCategory
``` C
static void OBJC_CATEGORY_SETUP_$_XYTree_$_XYTreeCategory(void ) {
    _OBJC_$_CATEGORY_XYTree_$_XYTreeCategory.cls = &OBJC_CLASS_$_XYTree;
}
```

### _OBJC_$_CATEGORY_XYTree_$_XYTreeCategory
``` C
static struct _category_t _OBJC_$_CATEGORY_XYTree_$_XYTreeCategory __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
    "XYTree",
    0, // &OBJC_CLASS_$_XYTree,
    (const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_XYTree_$_XYTreeCategory,
    0,
    0,
    (const struct _prop_list_t *)&_OBJC_$_PROP_LIST_XYTree_$_XYTreeCategory,
};
```

