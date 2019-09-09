---
layout: post
title: "行走在 Objective-C 动态类型"
date: 2013-03-12T13:58:00
draft: false
comments: true
published: true
toc: false
categories: [
    "iOS",
    "Tech"
]

tags: [
    "iOS",
    "Objective-C",
    "tech"
]
---

用了这么长时间的 Objective-C 语言, 怎么来"肤浅"描述下什么是 Objective-C 呢?, Objective-C 是一个 C 语言的超集, 整个语言是在 C 语言的基础上添加了一层预处理机制和强大运行时系统. 说到运行时系统, 整个语言精华就在这个这里,
它给整个语言增加了强大的且功能齐全的面向对象的编程接口, 函数式编程环境和神奇的动态类型系统.

今天我们就简单聊下 Objective-C 的动态类型, 废话不多说, 我们就说下体现这个特性其中三个方面.
<!-- more -->

* Object Introspection (__对象在运行时获取其类型的能力称为Introspection__)
* Packaging static C types with NSValue
* Bridging with Core Foundation objects

## Object Introspection

自省 (__Introspection__) 是对象揭示自己作为一个运行时对象的详细信息的一种能力, 例如确定一个 Objective-C 的对象类型, 通过 `isKindOfClass` 方法便可轻松确定.

``` objc
- (void)testObjectType:(id)obj
{
    if ([obj isKindOfClass: [NSNumber class]]) {
	// do something with number...
    } else if ([obj isKindOfClass: [NSValue class]) {
	// do something with values...
    }
}
```

为什么需要这个类型自省机制呢? 在一个应用程序中需要通过键值编码来判断一个对象类型, 例如 `Core Animation`的诸多动画属性...

* anchorPoint
* backgroundColor
* backgroundFilters
* borderColor
* borderWidth
* bounds
* compositingFilter
* contents
* contentsRect
* cornerRadius
* doubleSided
* filters
* frame
* hidden
* mask
* masksToBounds
* opacity
* position
* shadowColor
* shadowOffset
* shadowOpacity
* shadowRadius
* sublayers
* sublayerTransform
* transform
* zPosition

看似这么多属性, 其实这些属性类型仅仅包括 `CGPoint`, `CGRect`,
`CGFloat`, `CGImageRef`, `CGColorRef` 和 `BOOL` 这几种. 每种类型都有自己的类型值实现. 幸运是的 Objective-C 语言的动态类型系统允许我们传递一个泛型类型 `id`, 并在运行时再来确定这个泛型类型的具体类型, `id`仅仅是一个 `void` 指针. Objective-C 的对象这个结构体中具有一个 isa 指针, 指向定义这个对象的类对象.这个类对象包含了 Objective-C 对象的方法调度表, 属性和协议等信息, 这也是 Objective-C 动态能力的根源.

`NSObject` 可以简单看作为这样:

``` objc
@interface NSObject <NSObject> {
     Class    isa;
}
```
同样用 C 语言结构体可表示为:

``` objc
struct NSObject{
 　　Class isa;
 }
```
Class 的 typedef 定义:

``` objc
typedef struct objc_class *Class;
```
这样话 `NSObject` 可以看作为:

``` objc
struct NSObject{
　　objc_class *isa
}
```
再进一层看下 `objc_class` 便可直观的知道 `isa` 最终指向的信息:

``` objc
struct objc_class {
     Class isa;
     Class super_class;
     const char *name;
     long version;
     long info;
     long instance_size;
     struct objc_ivar_list *ivars;
     struct objc_method_list **methodLists;
     struct objc_cache *cache;
     struct objc_protocol_list *protocols;
 }
```
想进一步了解 Objective-C 对象模型的可以出门右转, 看下 @唐巧 写的[Objective-C对象模型及应用][Objective-C对象模型及应用].

## Packaging static C types with NSValue

我们之前说了 Objective-C 仅仅是对 C 语言添加了一层运行时皮肤, 所以经常看到一些底层实现大量使用了 `int`, `float`, `pointer` 和 `struct`等 C 基础数据类型. 然而这些基础数据类型似乎违反了 Objective-C 的动态类型特性.幸运是, Apple 提供了 NSValue 容器来包装这些基础数据类型, 它可以容纳任何 C 类型, 例如 `int`, `float`,`char`, `pointer`, `struct`等. 它不仅包含这些基础数据类型作为一个 Objective-C 对象, 并且还包含了这些类型的原始信息.

创建一个 `NSValue` 对象, 你需要传递一个数据的指针, 以及由 `@encode()` 生成的数据类型信息.

``` objc
CGPoint origin = CGPointMake(0,0);
NSValue * originValue = [NSValue valueWithBytes:&origin objCType:@encode(CGPoint)];
```

简单的来说 `@encode()` 就是可以把 Objective-C 的数据类型, 甚至是自定义类型, 函数或方法的元类型表示为 C 字符串(const char*),
关于 `Type encodings` 可以看看 Apple 的[Objective-C runtime type encodings][objc type].

举上几个栗子瞧瞧:

``` objc
@encoding(int ** )
// ==> "{^^i}"
@encoding(CGPoint)
// ==> "{CGPoint=ff}"
@encoding(CGColorRef)
// ==> "^{CGColor=}"
@encoding(NSObject)
// ==> "{NSObject=#}"
```

确定这种类型编码, 我们仅仅通过几步便可确定:

``` objc
if ([obj isKindOfClass:[NSValue class]]) {
    NSValue* value = (NSValue*) obj;
    if strcmp([value objCType], @encode(CGPoint)) == 0) {
	CGPoint origin;
	[value getValue:&origin];
    }
}
```

### UIKit addition to NSValue

Apple为我们提供了 `UIKit Category`, 可以以简便的方法来创建一个 `NSValue`, Link:[Category for NSValue][uikit nsvalue]

## Bridging with Core Foundation objects

虽然 `NSValue` 可以帮我们包装大部分的数据类型. 但是[Core Foundation][Core Foundation]中的`CGColorRef`, `CGImageRef` 等数据类型 `NSValue` 是无法对其进行数据包装,
这些数据类型需要通过 [toll-free briding][toll free] 来搞定. 关于 `toll-free briding` 可以看看我的另一篇 [Objective-C ARC下的陷阱和最佳实践][Objective-C ARC下的陷阱和最佳实践].

一个 `Core Foundation` 的数据类型其实也可以用类似 `id` 的形式表示. 通过 `CFTypeRef` 来表式未知的数据类型, 然后通过提供的C函数 `CFGetTypeID` 便可返回相应数据的真实类型.

``` objc
if (CFGetTypeID((__bridge CFTypeRef)obj), == CGImageGetTypeID()) {
    CGImageRef imgRef = CFBridgingRetain(obj);
    // do something
    CFRelease(imgRef);
}
```

一个 `CFTypeRef` 类似被标记为我们熟悉的 `id` 类型, 它能接受像 `isKindOfType:` 的 Objective-C 消息.
因此想判断出一个 `id` 类型的对象是相当安全的, 只要是 `NSObject`, `NSValue`, `CFTypeRef`, `CGColorRef` 等其他 Objective-C 对象或 Core Foundation 对象.


## References

* [how to test if an arbitary pointer is a valid NSObject][nsobject]
* [Objective C type encoding][objc type]
* [Inspecting core foundation object][Core Foundation]


[objc type]: https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100
[uikit nsvalue]: http://developer.apple.com/library/ios/#DOCUMENTATION/UIKit/Reference/NSValue_UIKit_Additions/Reference/Reference.html
[toll free]: http://developer.apple.com/library/ios/#documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/tollFreeBridgedTypes.html
[nsobject]: http://www.cocoawithlove.com/2010/10/testing-if-arbitrary-pointer-is-valid.html
[Core Foundation]: http://developer.apple.com/library/mac/#documentation/CoreFoundation/Conceptual/CFDesignConcepts/Articles/Inspecting.html
[Objective-C ARC 下的陷阱和最佳实践]:http://youngshook.com/post/ObjectiveC-ARC-Pitfalls/
[Objective-C 对象模型及应用]:http://blog.devtang.com/blog/2013/10/15/objective-c-object-model/
