---
layout: post
title: "用 Runtime 动态增加 Category 的属性"
date: 2014-07-14T16:12:00
draft: false
comments: true
published: true
toc: false
categories: [
    "iOS",
    "Tech"
]
tags: [
    "Objective-C",
    "tech"
]
---

### 前言
为什么要写个前言呢, 我想先谈下__[DRY][DRY] (Don't Repeat Yourself)__, 在经历过诸多软件项目开发后,相信大家每到一个时期都会心想, "尼玛,看来又要重构了!", 总是对自己开发的软件都有或多或少有不满意的地方,也许每次在新项目开始前, 心中都默念"DRY! DRY! DRY!", 但收效甚微. 懂得 DRY 并不代表会正确使用 DRY 来设计和实现, 因为这需要我们大量的实践和总结, 才能练好 DRY 实践的内功,今天我们就来使用 DRY 思想来解决一个实际问题. 这是 DRY 思想在 __[wikipedia][DRY]__ 的解释.
<!-- more -->

### 问题
在一个项目中每次重复写一段代码, 内心都有无数个草泥马在奔腾, 因此每当我发现有大量的重复的代码在一个项目中,我都有重构的冲动.
举个栗子, 在开发过程中经常会通过添加 Category 给 Class 添加一组 Property, 这要感谢伟大的 Runtime 给我提供的 `objc_getAssociatedObject` 和 `objc_setAssociatedObject` 关联引用方法.但是我发现每次有同样需求的时候总要写一遍一样的 getters/setters 方法.

### 如何解决
下面是在一个已存在的 Class 添加一组本身不存在的 Property 的 Category 例子.

``` objc
@interface Superman (YSKit)
@property (strong, nonatomic) UIColor *ys_ShirtColor;
@property (strong, nonatomic) NSArray *ys_Weapons;
@end
```
在什么时机动态的声明和实现我们要加的 Property 呢? 从解决问题目的出发, 会一个比较好的方案是在 Runtime 在执行的`+Load` 方法的时候.

``` objc
@implementation Superman (YSKit)
@dynamic ys_ShirtColor, ys_Weapons;

+ (void)load
{
  // 实现所有 Property 的 getters/setters 方法
  [self implementDynamicPropertyAccessors];
  // 或通过传入 Property 名字来实现其 Property 的 getters/setters 方法
  [self implementDynamicPropertyAccessorsForPropertyName:@"ys_Weapons"];
  // 或通过传入 Property 的正则表达式来实现其 Property 的 getters/setters 方法
  [self implementDynamicPropertyAccessorsForPropertyMatching:@"^ys_"];
}

@end
```

这样做看起来不错, 很方便, 可以省去大量给 Category 添加属性的代码, 避免了重复的代码. __Apple 的文档告诉我们原有 Class 的 `+Load` 方法执行完才会执行 Category 的 `+Load` 方法,而且有一个重要的点是 Category 的 `+Load` 方法不像其他方法一样, 并不会覆盖原有 Class 的 `+Load` 方法.__

可能发现 `+ (void)implementDynamicPropertyAccessors` 方法便能满足我们动态实现 Property 的需求,但是如果我们 Class 有不希望生成 getters/setters 方法的 Property, 便可通过另外另外两个方法做过滤来满足我们需求.

### 如何实现

所有的黑魔法都发生在 `+ (void)[NSObject implementDynamicPropertyAccessors]` 方法中,通过我们添加一个 NSObject 的 Category.

这个类方法的实现很简单:

``` objc
+ (void)implementDynamicPropertyAccessors
{
  [self enumeratePropertiesWithBlock:^(objc_property_t property){
    [self implementAccessorsIfNecessaryForProperty:property];
  }];
}
```

这个方法仅仅是遍历 self 的所有属性, 并转发到我们定义的另一个方法中做处理.

`+ (void)implementAccessorsIfNecessaryForProperty:` 将会判断所有转发进来的属性是否被声明为 Dynamic, 如果是将会为其实现 getters/setters 方法.

``` objc
+ (void)implementAccessorsIfNecessaryForProperty:(objc_property_t)property
{
  // 1.
  NSArray *attributes = [self attributesOfProperty:property];
  BOOL isDynamic = [attributes containsObject:@"D"];
  if (!isDynamic) {
    return;
  }

  // 2.
  BOOL isObjectType = YES;
  NSString *customGetterName;
  NSString *customSetterName;

  for (NSString *attribute in attributes) {
    unichar firstChar = [attribute characterAtIndex:0];
    switch (firstChar) {
      case 'T': isObjectType = [attribute characterAtIndex:1] == '@'; break;
      case 'G': customGetterName = [attribute substringFromIndex:1]; break;
      case 'S': customSetterName = [attribute substringFromIndex:1]; break;
      default: break;
    }
  }
  if (!isObjectType) {
    return;
  }

  // 3.
  static const void *key = &key;
  key++;

  // 4.
  const char *name = property_getName(property);
  [self implementGetterIfNecessaryForPropertyName:name customGetterName:customGetterName key:key];

  BOOL isReadonly = [attributes containsObject:@"R"];
  if (!isReadonly) {
    [self implementSetterIfNecessaryForPropertyName:name customSetterName:customSetterName key:key attributes:attributes];
  }
}
```

1. __首先我们判断转发进来的属性是否被声明为 Dynamic, 如果不是则直接返回, 无需为其实现 getters/setters 方法.__
2. __通过判断转发进来的属性的类型是否是 Objective-C 对象, 如果不是则直接返回.__
3. __接下来创建一个静态对象, 用于接下来的关联引用的 Key. (为了防止指针地址相同,通过自增的方式来满足每次生成的Key指针都不同)__
4. __最后我们通过属性的名称来增加 getters/setters 方法, 并且判断是否为 Readonly,如果是的话将仅仅生成 getters 方法.__

下面是最终通过 Runtime 增加 getters/setters 的方法实现:

__生成 `Getter` 方法:__

通过使用传入的 propertyName 或自定义的 `Getter` 方法名, 来生成 `Getter` 的 SEL, 然后通过 Block 创建`Getter` 的方法实现, Block 返回通过 Runtime 的关联引用方法得到 property 的 `iVar` 变量.

``` objc
+ (void)implementGetterIfNecessaryForPropertyName:(char const *)propertyName customGetterName:(NSString *)customGetterName key:(const void *)key
{
  SEL getter = NSSelectorFromString(customGetterName ?: [NSString stringWithFormat:@"%s", propertyName]);
  [self implementMethodIfNecessaryForSelector:getter parameterTypes:NULL block:^id(id _self) {
    return objc_getAssociatedObject(self, key);
  }];
}
```
__生成 `Setter` 方法:__

与 `Getter` 方法相比增加了判断 Property 的属性访问类型, 用来设置关联引用的策略.

``` objc
+ (void)implementSetterIfNecessaryForPropertyName:(char const *)propertyName customSetterName:(NSString *)customSetterName key:(const void *)key attributes:(NSArray *)attributes
{
  BOOL isCopy = [attributes containsObject:@"C"];
  BOOL isRetain = [attributes containsObject:@"&"];
  objc_AssociationPolicy associationPolicy = isCopy ? OBJC_ASSOCIATION_COPY : isRetain ? OBJC_ASSOCIATION_RETAIN : OBJC_ASSOCIATION_ASSIGN;
  BOOL isNonatomic = [attributes containsObject:@"N"];
  if (isNonatomic) {
    objc_AssociationPolicy nonatomic = OBJC_ASSOCIATION_COPY_NONATOMIC - OBJC_ASSOCIATION_COPY;
    associationPolicy += nonatomic;
  }

  SEL setter = NSSelectorFromString(customSetterName ?: [NSString stringWithFormat:@"set%c%s:", toupper(*propertyName), propertyName + 1]);
  [self implementMethodIfNecessaryForSelector:setter parameterTypes:"@" block:^(id _self, id var) {
    objc_setAssociatedObject(self, key, var, associationPolicy);
  }];
}
```
__通过 Runtime 函数增加方法:__

``` objc
+ (void)implementMethodIfNecessaryForSelector:(SEL)selector parameterTypes:(const char *)types block:(id)block
{
  BOOL instancesRespondToSelector = [self instancesRespondToSelector:selector];
  if (!instancesRespondToSelector) {
    IMP implementation = imp_implementationWithBlock(block);
    class_addMethod(self, selector, implementation, types);
  }
}
```
__在 Category Class 执行 `+Load` 时为每个传进来的 Property 遍历和执行每个 Block:__

``` objc
+ (void)enumeratePropertiesWithBlock:(void(^)(objc_property_t property))block
{
  NSParameterAssert(block);
  uint count = 0;
  objc_property_t *properties = class_copyPropertyList(self, &count);
  for (uint i = 0; i < count; i++) {
    objc_property_t property = properties[i];
    block(property);
  }
  free(properties);
}
```
__返回 Property 所有的 `attribute`. 用于做 Property 类型过滤:__

``` objc
+ (NSArray *)attributesOfProperty:(objc_property_t)property
{
  return [[NSString stringWithCString:property_getAttributes(property) encoding:NSUTF8StringEncoding] componentsSeparatedByString:@","];
}
```

下面是两个用于实现部分 Property 的动态访问, 与上面实现大体相同.

``` objc
+ (void)implementDynamicPropertyAccessorsForPropertyName;
+ (void)implementDynamicPropertyAccessorsForPropertyMatching;
```

__可以在[Github][Category]上找到本文 Category 的完整实现.__




[DRY]:http://en.wikipedia.org/wiki/Don't_repeat_yourself
[Category]:https://github.com/youngshook/YSDynamicProperties
