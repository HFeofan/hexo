---
title: Objective-C Nullability
date: 2018-03-14 10:09:57
tags: iOS
---

在使用OC开发的时候，常常会因为给一个方法传了一个为nil的参数导致运行时崩溃。在Swift中采用了Optional Type去规避这种问题，在编译时就给出警告。

# Swift

Swift中一个值是否可以为nil都有明确的声明，如 `var view1: UIView`, 这时不能给view1赋值为nil，`var view2: UIView?`，这样的话view2可以为nil，如果`view1 = view2`，这时会给出编译错误。

```swift
func printPersonName(name: String) -> Void {
    print(name);
}
  
//1
var name = "hahah";
name = nil; //Nil cannot be assigned to type 'String'
//2
let name: String? = "hahah";
printPersonName(name: name); //Value of optional type 'String?' not unwrapped; did you mean to use '!' or '??'
```
如上2，如果需要把optional赋给non-optional，需要手动强制解包(!)，或者使用(??)运算符。

## Optional Type

```swift
enum Optional<Wrapped> {
    case none
    case some(Wrapped)
}
```
optional 其实是一个enum。为nil时的值是none。

```swift
let shortForm: Optional<Int> = Int("42")
let number: Int? = Optional.some(42)
let noNumber: Int? = Optional.none
```

# Objective-C

```
@interface MAYStudent : NSObject
@property (nonatomic, strong) NSString *name;
@end
  
//Objective-C
MAYStudent *student = [MAYStudent new];
NSAttributedString *attributedString = [[NSAttributedString alloc] initWithString:student.name]; //没警告，name的类型为NSString *
  
//Swift
let student = MAYStudent()
let attributedString = NSAttributedString(string: student.name) //没警告， name的类型为String!
```
使用OC声明了一个Student的类，分别在OC和Swift中使用，编译器并不会给出警告，但是由于name可能会nil，所以这段代码并不安全。

Xcode6.3之后开始支持OC的nullability annotations。有了这个特性之后可以给变量添加nullability声明。指示一个变量是否可以为nil。

## nullability annotations

### \_Nullable & \_Nonnull
这个两个是核心的声明语句，`_Nullable`表示指针可能为nil或者NULL，`_Nonnull`表示不能，这个修饰的是指针，所以在需要靠近指针变量，即在星号(`*`)之后。示例：

```objectivec
@interface MAYStudent : NSObject
@property (nonatomic, strong) NSString * _Nonnull name;
@property (nonatomic, copy) void (^ _Nullable testBlock)(NSString * _Nonnull, NSError * _Nullable);
 
- (NSString * _Nullable)email;
- (void)addItem:(NSString * _Nonnull)item;
 
@end
```

### nullable & nonnull

`_Nullable` 和 `_Nonnull`可以在任何使用C常量的地方使用，基本上是所有地方都可以用。不过在方法和属性声明中提供了更加友好的写法`nullable`和`nonnull`。示例：

```objectivec
@interface MAYStudent : NSObject
 
@property (nonatomic, strong, nonnull) NSString *fullName;
@property (nonatomic, copy, nullable) void (^test1Block)(NSString * _Nonnull, NSError * _Nullable);
 
- (nonnull NSString *)myFullName;
- (void)otherAddItem:(nonnull NSString *)item;
 
@end
```
block中的参数还是需要使用`_Nonnull` 和`_Nullable`修饰。

### Audited Regions

如果头文件中部分使用了nullability annotations，编译器会对其它没有使用的地方显示警告。这样警告就会很多，为了更方便的使用nullability annotations，可以使用`NS_ASSUME_NONNULL_BEGIN`和`NS_ASSUME_NONNULL_END`将类的声明包起来。示例：

```objectivec
NS_ASSUME_NONNULL_BEGIN
@interface MAYStudent : NSObject
@property (nonatomic, strong) NSString * name;
@property (nonatomic, copy) void (^testBlock)(NSString *, NSError *);
 
- (NSString * _Nullable)email;
- (void)addItem:(NSString *)item;
 
@end
NS_ASSUME_NONNULL_END
```

这个宏只有NONULL，对于需要nullable的需要单独指定。

使用audited regions需要注意一下几点：

1，typedef中的类型的nullability需要更具上下文判断。即使typedef本身的定义在Audited Regions中，但是在使用的地方不一定是nonnull的。

假如用`typedef`定义了一个block，如 `typedef void(^MyBlock)(NSString *name)`， 这个定义如果在Audited Regions中，那么他的参数就会被指定为nonnull。MyBlock类型的变量需要根据上下文判断。

```objectivec
//1
NS_ASSUME_NONNULL_BEGIN
 typedef void(^MyBlock)(NSString *name);
@interface MAYStudent : NSObject
@property (nonatomic, copy) MyBlock block; // block 这个变量是nonnull的，参数name也是nonnull的。
@end
NS_ASSUME_NONNULL_END
  
//2
@interface MAYTeacher : NSObject
@property (nonatomic, copy) MyBlock block;  // block 这个没有指定nullability, 但是它的参数name是nonnull的。
@end
```

2，复杂的指针类型，如`id *`，需要显示指定。比如，指定一个non-nullable的指针指向一个nullable的对象：`_Nullable id * _Nonnull`(苹果给的例子，个人觉得应该是 `id _Nullable * _Nonnull`， 即`_Nullable` 在`id`之后)。
不能在属性中声明`id *`的属性，作为方法的参数可以。不指定nullability会有警告。

![nullability1](/images/ocnullability1.png)

3，特殊的类型如`NSError **`，通常会假定为nullable的指针指向nullable的NSError引用。

```objectivec
NS_ASSUME_NONNULL_BEGIN
@interface MAYStudent : NSObject
@property (nonatomic, strong) NSString * name;
 
- (void)modifyName:(NSString *)name error:(NSError **)error;
 
@end
NS_ASSUME_NONNULL_END
```

在调用的时候代码提示的nullability为：
![nullability2](/images/ocnullability2.png)

### Compatibility

`_Nullable` 和 `_Nonnull` 是Xcode7新的语句，替换xcode6.3开始的`__nullable`和`__nonnull`。说是为了避免和第三方库潜在的冲突。

# Usage

有了nullability annotations之后，OC类在Swift中使用应该可以确保一些安全，因为可选类型和非可选类型是两种类型，在赋值时需要手动转换。但是在OC中却没有这么严格的检查。给nonnull的变量赋值nil只会产生警告而不是错误，而且是直接赋值时才有警告。如果是一个变量赋给nonnull的值，都不会有警告。所以对于不需要nil的情况，nonnull并不能万全保证，还需要手动判断。

![ocnullability3](/images/ocnullability3.png)


参考：https://developer.apple.com/swift/blog/?id=25