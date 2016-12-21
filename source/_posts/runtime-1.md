---
title: 运行时前篇——对象初始化设置分析
date: 2016-12-18 10:22:38
tags: iOS
---

在项目中通常使用一些运行的方法做一些hack，或者在开发一个库时使用运行时解耦。这些操作涉及到Objective-C的对象模型，本篇内容会讨论OC的对象的一些初始化设置。附ojbc的源码：https://opensource.apple.com/tarballs/objc4/

这里需要借助`clang -rewrite-objc`这个命令，它可以把OC代码转成C/C++代码。

示例代码：
```objectivec
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface XYTree : NSObject

@property (nonatomic, strong) NSString *xyName;

- (void)doTask;
- (void)doOtherTask;

+ (void)classMethodDoTask;

@end

@interface XYTree ()

@property (nonatomic, strong) NSString *exName;

@end

@implementation XYTree

- (void)doTask
{
    
}

+ (void)classMethodDoTask
{
    NSLog(@"classMethodDoTask");
}

@end

@interface XYTree (XYTreeCategory)

@property (nonatomic, strong) NSString *categoryName;

+ (void)categoryClassMethodDoTask;

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

+ (void)categoryClassMethodDoTask
{
    NSLog(@"categoryClassMethodDoTask");
}

@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        XYTree *object = [XYTree new];
        [XYTree classMethodDoTask];
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
```C
XYTree *object = ((XYTree *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("XYTree"), sel_registerName("new"));
```
浏览生成的main.cpp，发现了两个inithook
```C
__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CLASS_SETUP[] = {
    (void *)&OBJC_CLASS_SETUP_$_XYTree,
};

__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CATEGORY_SETUP[] = {
    (void *)&OBJC_CATEGORY_SETUP_$_XYTree_$_XYTreeCategory,
};
```
一个是class的setup，一个是category的setup。

## OBJC_CLASS_SETUP_$_XYTree
```C
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
```C
extern "C" __declspec(dllexport) struct _class_t OBJC_METACLASS_$_XYTree __attribute__ ((used, section ("__DATA,__objc_data"))) = {
    0, // &OBJC_METACLASS_$_NSObject,
    0, // &OBJC_METACLASS_$_NSObject,
    0, // (void *)&_objc_empty_cache,
    0, // unused, was (void *)&_objc_empty_vtable,
    &_OBJC_METACLASS_RO_$_XYTree,
};
```

### OBJC_CLASS_$_XYTree
```C
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

### 相关联的初始化

#### _OBJC_METACLASS_RO_$_XYTree
``` C
static struct _class_ro_t _OBJC_METACLASS_RO_$_XYTree __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    1, sizeof(struct _class_t), sizeof(struct _class_t), 
    (unsigned int)0, 
    0, 
    "XYTree",
    0, 
    0, 
    0, 
    0, 
    0, 
};
```
`XYTree`的metaclass的的`ro`没有什么特别的信息。

#### _OBJC_CLASS_RO_$_XYTree
```C
static struct _class_ro_t _OBJC_CLASS_RO_$_XYTree __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    0, __OFFSETOFIVAR__(struct XYTree, _xyName), sizeof(struct XYTree_IMPL), 
    (unsigned int)0, 
    0, 
    "XYTree",
    (const struct _method_list_t *)&_OBJC_$_INSTANCE_METHODS_XYTree,
    0, 
    (const struct _ivar_list_t *)&_OBJC_$_INSTANCE_VARIABLES_XYTree,
    0, 
    (const struct _prop_list_t *)&_OBJC_$_PROP_LIST_XYTree,
};
```
`XYTree`的`ro`包含了我们声明的method，ivar和property。

#### _OBJC_$_INSTANCE_METHODS_XYTree
``` C
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
看上边的基本结构里没有`struct _method_list_t`这个结构体的声明。在main.cpp中找到了直接使用该结构初始化的方法列表。可以看到，已经帮我们自动生成了属性的getter和setter方法。值得注意的是在示例代码中还声明了`doOtherTask`方法，没有实现它，这个结构里也没有出现这个方法。

在上述代码中也声明并实现了一个类方法`classMethodDoTask`,在`struct _class_ro_t`里并没有看到class method的存储位置，在main.cpp中也没有看到类方法的相关信息。

#### _OBJC_$_INSTANCE_VARIABLES_XYTree
```C
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

#### _OBJC_$_PROP_LIST_XYTree
``` C
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

与ivar list对比，这里只有`xyName`一个property。


## OBJC_CATEGORY_SETUP_$_XYTree_$_XYTreeCategory
```C
static void OBJC_CATEGORY_SETUP_$_XYTree_$_XYTreeCategory(void ) {
    _OBJC_$_CATEGORY_XYTree_$_XYTreeCategory.cls = &OBJC_CLASS_$_XYTree;
}
```

### _OBJC_$_CATEGORY_XYTree_$_XYTreeCategory
```C
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
`struct _category_t`没有`ivars`字段，所以不能给category增加ivar。`struct _category_t`里有`class_methods`字段，但是没在main.cpp中看到类方法的信息。

### 相关联的初始化

#### _OBJC_$_CATEGORY_INSTANCE_METHODS_XYTree_$_XYTreeCategory
``` C
static struct /*_method_list_t*/ {
    unsigned int entsize;  // sizeof(struct _objc_method)
    unsigned int method_count;
    struct _objc_method method_list[2];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_XYTree_$_XYTreeCategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_objc_method),
    2,
    {{(struct objc_selector *)"setCategoryName:", "v24@0:8@16", (void *)_I_XYTree_XYTreeCategory_setCategoryName_},
    {(struct objc_selector *)"categoryName", "@16@0:8", (void *)_I_XYTree_XYTreeCategory_categoryName}}
};
```
同样的，自动为`categoryName`生成了getter和setter方法。

#### _OBJC_$_PROP_LIST_XYTree_$_XYTreeCategory
``` C
static struct /*_prop_list_t*/ {
    unsigned int entsize;  // sizeof(struct _prop_t)
    unsigned int count_of_properties;
    struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_XYTree_$_XYTreeCategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
    sizeof(_prop_t),
    1,
    {{"categoryName","T@\"NSString\",&,N"}}
};
```

# 类方法

了解到上边的信息之后，产生一个疑问，为什么没有看到类方法的相关信息。看相关文章说类方法存在metaclass的methodlist里，但是看上边`_OBJC_METACLASS_RO_$_XYTree`的`baseMethods`并没有值。这里先留下个疑问。

# category的加载

对于category的加载可以看美团点评博客上的[深入理解Objective-C：Category](http://tech.meituan.com/DiveIntoCategory.html)，写的很详细。





