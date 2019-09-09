---
layout: page
title: "换个姿势调试 UIWebView"
date: 2013-04-06T22:32:00
draft: false
comments: true
published: true
toc: false
categories: [
    "iOS",
    "Tech"
]
tags: [
    "webview",
    "tech"
]
---

Hybrid App 开发 App 现在越来越流行, 尤其对于快速迭代开发来说, Hybrid App 是一个快速验证需求点的好方式,在 Hybrid App 过程中不可避免的要和 UIWebView 打交道, 调试 `UIWebView` 上的网页不像桌面版的 Safari 或 Chrome 一样有强大的开发者工具, 在手机上调试 `UIWebView` 是一件痛苦的事情, 找了几种调试的方式, 对比下来发现还是麻烦, 不如换个姿势找找其他方式?

当前有几种调试 UIWebView 的方式:

__1.通过 `NSProxy` 来 Hook UIWebView 的事件.__

__2.通过私有 API 来开启远程调试__

第一种方式可能相对复杂不便于进行全局调试, 有兴趣的同学可以看看这篇文章 [NSProxy UIWebView][NSProxy UIWebView] 我们就简单说下如何通过私有 API 来调试 `UIWebView` 的方法.
<!-- more -->
通过对 `WebKit.framework` 直接 `class-dump` 我们可以得到 `UIWebView` 的私有API.

``` objc
@interface WebView (WebPrivate)
+ (void)_enableRemoteInspector;
@end
```
通过方法名还是很容易看出来这是我们想要的私有 API__(Available iOS5)__, 那么通过 `+[WebView _enableRemoteInspector]` 我们来开启远程调试.

``` objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

    [NSClassFromString(@"WebView") _enableRemoteInspector];

}
```

当然这个私有 API 不可能通过编译器的检查, 所以我们使用其他方式绕过编译器的检查, 通过 `performSelector:@selector()` 方法来绕过消息检查.

``` objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

#if (TARGET_IPHONE_SIMULATOR)
    [NSClassFromString(@"WebView") performSelector:@selector(_enableRemoteInspector)];
#endif

}
```

OK, 仅仅通过一个私有 API 便可进行远程 Debug, 现在打开模拟器然后在 Safari 地址栏输入 `localhost:9999` 便可.

不想折腾的同学还也可以看看 [brainlock][brainlock] 的开源项目 [ios-remote-inspector][ios-remote-inspector], 更多的思路永远不是坏事.


[NSProxy UIWebView]:http://blog.fenrir-inc.com/jp/2013/11/nsproxy.html
[brainlock]:https://github.com/brainlock
[ios-remote-inspector]: https://github.com/brainlock/ios-remote-inspector
