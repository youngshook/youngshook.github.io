---
title: 提升 76% 的编译速度！Coupang App 工程提效实践
date: 2019-09-24 11:11:34
draft: false
published: true
tags: []
---

<a name="OzlC8"></a>
# 一.前言

Coupang 作为全球最大、增长最快的电商公司之一，工程生产力直接影响着公司的持久创新力和公司在市场上的作为，随着业务的快速增长，作为直接面向消费者的 Coupang App 的代码量也快速增长，构建的时间也随之不断增加，构建时间直接影响着团队的工程生产力，当前仅 iOS App端的代码量 55W 行，每次完整打包一次的耗时平均在 90 分钟，最坏的情况下需要 2 个小时，这对于经常一天要发布内测版本两到三次的情况，会影响整个团队的开发进度。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565257690139-ffe34248-f6f0-456e-ac7b-bab9cb3a13e2.png#align=left&display=inline&height=334&name=image.png&originHeight=545&originWidth=627&size=130585&status=done&width=384)

本文将分析打包速度慢的瓶颈所在，以及应对的解决方案，最终仅在编译和工程上的优化, 成功将编译时间降低到 20 分钟，再通过升级 CI 硬件性能后，最终将持续集成打包降低至 9 分钟。

<a name="QmjdZ"></a>
# 二.当前项目架构

- **Swift:** 2656 个文件， 22W 行代码
- **Objective-C:** 1529 个文件，33W 行代码
- **子工程数量:** 22 个
- 使用 Cocoapods 管理第三方依赖
- 总代码行数 55W

<a name="L6yjN"></a>
# 三.解决方案
<a name="zY57Z"></a>
### 1.优化 Swift 编译时间
<a name="Ub3f9"></a>
####     Swift 代码编译耗时分析
由于项目的代码量比较大，首先目标就是要优化代码的编译速度，首先需要把耗时过长的文件找出来，然后进行重点优化，这里会在 OTHER_SWIFT_FLAGS 加入两个编译参数:<br />

  - -Xfrontend: 如果编译或类型检查时耗时多长，则在Xcode中输出警告。
  - -debug-time-function-bodies：输出每个函数的编译时长。<br />

然后通过使用 [BuildTimeAnalyzer](https://github.com/RobertGummesson/BuildTimeAnalyzer-for-Xcode) 工具分析 Xcode 编译 Log, 即可看到类似于下图的分析结果界面。

      ![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565256181268-d865bce5-6a1a-47bf-a513-95b68b04be9d.png#align=left&display=inline&height=319&name=image.png&originHeight=870&originWidth=1738&size=139375&status=done&width=638)

通过上面的分析结果，我们可以重点关注编译时间比较长的文件和函数，并通过以下方式优化 Swift 代码。

     ![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565256241834-43f407d4-abcb-443f-b420-1ad21ff7c18a.png#align=left&display=inline&height=321&name=image.png&originHeight=495&originWidth=1000&size=197672&status=done&width=649) 
<a name="00f5a5f0"></a>
###### 
<a name="6fuz5"></a>
####    为复杂的 Swift 属性使用明确的类型
在 Swift 中定义变量时可以不带类型声明，Xcode 编译器会更具初始化的值推断变量类型，这为 Swift 开发者提供了很大的便利，但这同时也增加的编译器的分析推断时间，从减少编译时间的角度考虑，对于相对复杂代码语句，减少编译器类型推断，可以大大减少 Xcode 的编译时间。<br />例如:<br />

```swift
// build time: 330ms
let tm1 = ceil(abs(PlayerWaitingStartInterval - timer.fireDate.timeIntervalSinceNow) * 1000)
// build time: 79ms
// Add type annotation
let tm1: Double = ceil(abs(PlayerWaitingStartInterval - timer.fireDate.timeIntervalSinceNow) * 1000)
// build time: 26ms
// Add type annotation & separate into two parts
let interval: Double = abs(PlayerWaitingStartInterval - timer.fireDate.timeIntervalSinceNow)
let tm1: Double = ceil(interval * 1000)
```


<a name="P32dT"></a>
####    拆解复杂的 Swift 表达式
对复杂表达式进行简化，简化的代码不仅可以提高编译效率，也具有更好的可读性，例如下面样例代码编译时间相差 30 倍.<br />

```swift
let sum = [1, 2, 3].map { String($0) }.compactMap { Int($0) }.reduce(0, +)
```

<br />to<br />

```swift
let numbers = [1, 2, 3]
let stringNumbers = numbers.map { String($0) }
let intNumbers = stringNumbers.compactMap { Int($0) }
let sum = intNumbers.reduce(0, +)
```


<a name="cuYsF"></a>
####    避免使用加号对字符串或数组进行拼接
对字符串或数组的拼接，在日常开发中十分常见，需要注意拼接方式对编译时间的影响，例如下面样例代码编译时间相差 20 倍.<br />对于 String 拼接:<br />

```swift
"abc" + stringA + "cbd"
```

<br />to<br />

```swift
"abc\(stringA)cbd"
```

<br />对于 Array 拼接:<br />

```swift
arrayA + arrayB
```

<br />to<br />

```swift
arrayA.append(contentsOf: arrayB)
```

<br />上面举例了几种对 Swift 编译时长影响较大的写法，优化后在代码层面编译时间有很大提升，同时缩短编译时长的代码规范还有很多，但可优化的时间不多，不再一一列举。

<a name="wUuaG"></a>
### 2.减少代码间的依赖
对于一个Swift/Objective-C混编项目来说，Objective-C Bridging Header 是 Objective-C 向 Swift 暴露的接口，Swift 生成的 *-Swift.h 代表的是 Swift 向 Objective-C 暴露的接口，相互桥接和引用过多的头文件，将造成每次编译时间的指数增长，使用以下方式来减少编译依赖。

  - 删除无用的头文件引用
  - 使用 Clang modules 技术，用 [@import ]() 来替代 #import。
  - 使用 Forward declaration, 用 [@class ]() 声明引用。
  - 尽可能的缩小访问控制范围, 使用 Static dispatch 代替 Dynamic dispatch。



<a name="ouPO8"></a>
### 3.预编译 Frameworks
CocoaPods 以源代码的形式集成第三方库，每次打包过程中这些第三方库都会重新编译，对于打包来说毫无意义，加快编译的一个方式是把第三方库全部预编译好，再集成到工程里，以减少不必要编译时间。<br />

  - 使用 [cocoapods-binary](https://github.com/leavez/cocoapods-binary) 预编译依赖库，直接使用二级制参与编译链接。
  - 裁剪第三方依赖库，即通过删除无用类、无用方法、重复方法等减少不必要编译的代码。

<a name="PVWef"></a>
### 4.Xcode 项目的调整

- **让 Xcode 显示编译时间**
```bash
$ defaults write com.apple.dt.Xcode ShowBuildOperationDuration -bool YES
```

![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565256304224-68fffe8b-0ae1-4aa1-ae6e-dcf8b0012ccc.png#align=left&display=inline&height=43&name=image.png&originHeight=65&originWidth=697&size=8462&status=done&width=464)

- **调整编译优化等级**<br />
编译优化等级的基本原理是牺牲编译时性能，追求运行时的性能，在编译时删除无用代码，保留调试信息，函数内联等，因此提升打包速度的秘诀就是反其道而行之，牺牲运行时性能来换取编译时性能。对于日常测试打包，可以将 Optimize level 设置为 -O0, 表示不做任何优化, 以提升编译器的编译速度。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565256318464-ee5829ac-e151-4f03-a590-1bd243da50ed.png#align=left&display=inline&height=600&name=image.png&originHeight=600&originWidth=1508&size=137159&status=done&width=1508)

- **关闭 Bitcode**<br />
BitCode 是 iOS 9 引入的新特性，是由 LLVM 引入的一种中间代码，当这个属性设置为 YES 的时候，Xcode 在打包的时候，会将项目编译成很多个设备对应的安装包，这样在编译打包的时候就比较耗时，但对于内部打包不需要上传 AppStroe, 因此我们可以关闭此特性以减少编译打包时间。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565256420178-c88af641-9b4b-490d-87d3-58bd46153bed.png#align=left&display=inline&height=278&name=image.png&originHeight=278&originWidth=1148&size=40508&status=done&width=1148)

- **关闭 dSYM 生成**<br />
在大部分场景下测试，并不需要生成 dSYM 文件，仅当在最终 App Release Test 和提交应用市场的时候开启即可，这可以为我们减少 40s 左右的打包时间。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565256406047-54157656-eec8-410f-af13-666ad90da681.png#align=left&display=inline&height=194&name=image.png&originHeight=194&originWidth=1434&size=46547&status=done&width=1434)

- **把 IO 操作映射到内存**<br />
在编译和最终的打包操作中 Derived Data 目录中会有大量的 IO 操作，通过建一个虚拟磁盘，将会把磁盘 IO 优化为 内存 IO，从而提高速度。

```bash
$ hdid -nomount ram://4194304
$ newfs_hfs -v DerivedData /dev/rdiskN
$ diskutil mount -mountPoint ~/Library/Developer/Xcode/DerivedData /dev/diskN
```

<a name="dCn4d"></a>
### 5.模块化代码
通过代码组件化，切断不同业务代码之间依赖，使得每次编译的时候就只需要编译自己模块下的代码。其他模块的代码将会被编译后缓存，不需要重复编译，从而减少编译时间。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/164293/1565256441747-18bfe5d1-4d8c-44f1-b315-ad7d16373382.png#align=left&display=inline&height=1972&name=image.png&originHeight=1972&originWidth=2586&size=215770&status=done&width=2586)

<a name="0Zk0Z"></a>
# 四.总结
通过应用上面的这些优化方案，很大程度的提升了 App 的打包速度，对于每天都要进行多次打包操作的工程师来说，更是节省了很多时间。在方案调研的过程中，研究过类似 Buck 、Bazel 、CCache等编译优化方案，但性能优化的路上没有银弹，只有根据项目实际情况选择合适的方法优化，才能达到如期的效果，希望本文可以为你提供一些思路。
