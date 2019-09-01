---
layout: post
title: "Objective-C ARC 下的陷阱与最佳实践"
date: 2013-05-22T19:45:00
draft: false
comments: true
categories: Objective-C
published: true
toc: false
---

Objective-C 是一个非常酷的编程语言 (尽管它和 C++ 一个年代出生). 它的流行不仅仅是因为 iOS 和 MacOS 平台,也是其在移动平台上强大的性能. 然而在日常开发过程中仍然要面对这个古老语言的诸多坑, 例如面对 Objective-C 的内存管理时, 虽然有强大的 ARC, 并且 Xcode 已经有很智能的警告提示, 但当面对复杂的内存管理的时候,仍然会掉进坑中, 例如在 ARC 环境下进行 `Core Foundation` 和 `Foundation` 之间数据类型交换. 这种技术也叫做 `Toll-Free Bridging`, 看起来很高端的样子, 今天我们就来说说如何以轻盈的姿势避开这些坑.
<!-- more -->

## ARC 的最佳实践

先说下在使用 ARC 时一些所有权修饰符用法, [@amattn][amattn]大神写的这篇[ARC 最佳实践][best practices].一篇关于 ARC 的最佳实践的文章, 下面是对部分的摘抄翻译.

* 如果需要保留对象的实例变量应该使用 `strong`.

``` objc
@property (nonatomic, strong) id childObject;
```

* 如果需要打破循环引用,应该使用 `weak`.

``` objc
@property (nonatomic, weak) id delegate;
@property (nonatomic, weak) NSTimer timer;
```

* 对于标量(非对象)的修饰使用 `assign`

``` objc
@property (nonatomic, assign) CGFloat width;
@property (nonatomic, assign) CGFloat height;
```

* 对于不可变的变量应使用 `copy`, 例如 NSString 和 Block. 避免使用在可变容器(例如 `NSMutableArray`.),如果修饰可变容器, 请使用 `strong`.

``` objc
@property (nonatomic, copy) NSString* name;
@property (nonatomic, copy) NSArray* components;
@property (nonatomic, copy) (void (^)(void)) job;
@property (nonatomic, strong) NSMutableArray* badPatterns;
```

* In dealloc
    - remove observers
    - unregister notifications

* IBOutlets 修饰的属性通常使用 `weak`, 除非是这个属性是在 nib 文件中是顶层的 File’s Owner 对象, 那么使用`strong`, 同时在使用完成后需要在
  `-(void)viewDidUnload` 中释放为 `nil`.


## ARC 所有权修饰符

1. `__strong` 是默认修饰符. 表示对对象的`强引用`. 持有强引用的变量在超出其作用域时被废弃, 随着强引用的失效, 引用的对象会随之释放.
2. `__weak` 修饰符的变量 (即弱引用) 不持有对象, 所以在超出其变量作用域时, 对象即被释放, 并且并被赋值为 `nil`.
3. `__unsafe_unretained` 修饰符正如其名 unsafe 所示, 是不安全的所有权修饰符, 在所有权上类似 `__weak`,但是并不会在超出其变量作用域时被赋值为 `nil`.
4. `__autoreleasing` 用来修饰一个声明为 (id *)的函数的参数, 当函数返回值时被释放.

需要注意的是ARC的所有权修饰符只能来修饰 `指针类型`, 也就是说你只能把所有权修饰符放在星号的右边.

``` objc
MyClass * __weak w_self = self;    // 正确
MyClass __weak * w_self = self;    // 错误! 会引起令人抓狂的Bug!
__weak MyClass * w_self = self;    // 错误!

__weak typeof(self) w_self = self;
// 正确,类似
// __weak (MyClass *) w_self = self;

typeof(self) __weak w_self = self; // 正确
```

可能在互联网上能看到很多的错误使用 ARC 修饰符. Apple 给出的 ARC 修饰符[范式][apple]:

> You should decorate variables correctly. When using qualifiers in an object
> variable declaration, the correct format is:
> `ClassName * qualifier variableName;`

## ARC 和 Toll-free bridging

在 ARC 使用过程中, 最容易出错的地方就是 `Core Foundation` 和 `Foundation` 之间数据类型交换.
按照下面这些规则使用, 我们就能容易的避免这些陷阱.

* Objective-C to CF, 需要 retain.
* CF to Objective-C, 需要 release.
* Core Foundation 下没有 autorelease, 所以你必须遵循 Core
  Foundation的命名规则:
  - 如果对象返回自从 `Create` 或 `Copy` 方法名开头的函数, 你将会拥有这个对象, 并且需要手动 release 这个对象.
  - 如果方法名包含 `Get`, 你不拥有此对象, 因此你不需要 release 这个对象.

有两种方法来保留一个 CF 对象: 使用类型转换 `(__bridge_retained)` 或者 C 方法 `CFBridgingRetain`,在Clang文档中推荐使用类型转换的方式, 我更喜欢后者, 因为更加便于阅读.

``` objc
CFArrayRef arr = CFBridgingRetain( @[@"abc", @"def", @3.14] );
// or CFArrayRef arr = (__bridge_retained CFArrayRef)@[...];
// do stuffs..
CFRelease(arr);
```

当你从包含 `Create` 或 `Copy` 方法名的 Core Foundation 方法中获取一个对象时, 应该使用 `(__bridge_transfer)` 或 `CFBridgingRelease` 来做转换.

``` objc
- (void)logFirstNameOfPerson:(ABRecordRef)person {
    NSString *name = (NSString *)CFBridgingRelease(ABRecordCopyValue(person, kABPersonFirstNameProperty));
    NSLog(@"Person's first name: %@", name);
}
```

## ARC 在 Toll-free bridging 中的陷阱

有时候我们会看到这样的代码:

``` objc
- (CGColorRef)foo {
    UIColor *color = [UIColor redColor];
    return [color CGColor];
}
```

当心, 这段代码可能随时引起程序崩溃, 因为我们不持有 UIColor, 它会在创建之后立即释放,从而所拥有的 CGColor 将被释放, 从而引发 Crash.
下面有三种方式来修复这段代码:

* 使用 `__autorelease` 类型修饰符. UIColor 将会在当前 RunLoop 结束时被释放.

``` objc
- (CGColorRef)getFooColor {
    UIColor *__autoreleasing color = [UIColor redColor];
    return [color CGColor];
}
```

* 使用类型转换为 Core Foundation 类型, 并改变对象持有者.

``` objc
- (CGColorRef)fooColorCopy {
    UIColor* color = [UIColor redColor];
    CGColorRef c = CFRetain([color CGColor]);
    return c;
}
.....
CGColorRef c = [obj fooColorCopy];
// do stuffs
CFRelease(c);
```

* 使用 self 拥有这个对象. 但是如果 self 被释放, 仍然会引起 Crash.

``` objc
- (CGColorRef)getFooColor {
    CGColorRef c = self.myColor.CGColor;
    return c;
}
```

## ARC 在 Block 中的陷阱

当你使用一个变量在 self 所持有的 Block 中时, 将会引起一个循环引用.

``` objc
@interface MyClass {
    id child;
}
@property (nonatomic, strong) (void(^)(void)) job;
@end

@implementation MyClass
- (void)foo {
    self.job = ^{
        [child work];
        // will expand to [self->child work]
    };
}
```

避免这种陷阱的一种方式是使用弱引用的 self. 并且在 Block 中强引用这个弱引用对象 self, 即形成所谓的`对外弱引用对内强引用`.
对内再次强引用的原因是弱引用对象可能随时被释放, 我们必须保证这个对外弱引用对象在使用时是有效的.

``` objc
- (void)foo {
    MyClass* __weak w_self = self;
    self.block = ^{
        MyClass* s_self = w_self; // self 被强引用,但仅仅在这个作用域中.
        if (s_self) {
            [s_self->child work];
            // do other stuffs
        }
    };
}
```

## ARC 在 NSError 中的陷阱

如果你要实现一个 NSError 的方法, 请务必使用正确的所有权修饰符.


``` objc
- (void)doStuffWithError:(NSError* __autoreleasing *)error; // 正确
- (void)doStuffWithError:(__autoreleasing NSError **)error; // 错误!
```

事实上, 当你创建一个 NSError 对象时, 最好声明为 `__autoreleasing`.

``` objc
NSError* __autoreleasing error = nil; // 正确
__autoreleasing NSError* error = nil; // 错误
NSError* error = nil;
```

[apple]: http://developer.apple.com/library/mac/#releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html
[amattn]: https://twitter.com/amattn
[best practices]: http://amattn.com/2011/12/07/arc_best_practices.html


