---
title: React Native 拆包加载
date: 2023-10-28 14:31:00
categories:
  - Front end
  - RN
tags:
  - RN
  - React Native
  - React
  - JavascriptCore
  - JSI
  - Split
  - loading
description: React Native 拆包打包及加载逻辑、加载流程
---

## 了解js在内存中是什么样的

现在我们有了一个基础包和多个业务包， 但我们该如何去加载这些RN包呢？首先我们要看一下js加载之后是什么样的呢。我们可以通过safari或safari technology preview的开发者菜单查看jscontext。

<image src="../assets/rn-6.png"/>

我们发现， 加载完成后就是将指定文件目录读进了内存，那我们需要怎么样才能把这个文件读进内存呢？

方案一、原生端读出来js文件，通过通知传递给js端， js端通过eval来执行该代码。

方案二、看原生端RCTBridge是如何加载js代码的，我们也通过这种方法加载。

## 把拆包后的代码加载到内存

不用选择，肯定是方案二更靠谱一些。于是我们找到了这么两个方法。

```objective-c
- (void)executeSourceCode:(NSData *)sourceCode sync:(BOOL)sync;
- (void)executeSourceCode:(NSData *)sourceCode bundleUrl:(NSURL *)bundleUrl sync:(BOOL)sync;
```

有什么区别呢？ 从参数可以看出一个没有设置jsbundle的路径，一个提供了参数设置该路径。我们选择多一个参数的方法来使用吧。

于是我们使用刚才打包出的js包，通过这两个方法来加载。得到如下加载结果。

<image src="../assets/rn-7.png"/>

我们成功把两个RN包加载到了内存里。

## 遇到的问题

太顺利了，怎么可能这么顺利就完成了？ 

其实这里面还有其他注意事项：

### 基础包要先加载完成

基础包是业务包运行的基石，一定要保证基础包加载完成后才能去加载业务包， 不然会有很多异常。那怎么保证基础包加载完成了呢？ 

1. 上面提到过一个js代码加载完成的通知： RCTJavaScriptDidLoadNotification。

2. 通过判断RCTBridge的isLoading属性是否变为了false。

```objective-c
 while (gRCTBridge.isLoading) {//侦听基础包是否加载完成 阻塞后续逻辑
      }
```

### 在js线程中执行代码

代码的执行一定要在js线程中进行， 不然会有一些莫名其妙的崩溃问题。

```objective-c
[gRCTBridge.batchedBridge dispatchBlock:^{
      [gRCTBridge.batchedBridge executeSourceCode:source.data bundleUrl:bundUrl sync:YES];
 } queue:RCTJSThread];
```

### 先加载完js代码，再创建RCTRootView

首先要保证js代码加载完成了，再去创建RCTRootView，不然会抛出require unknown module的异常。

### 拆包的好处

做了这么多，拆包到底有哪些优点值得我们去做呢？

- 拆分业务逻辑，减少业务耦合度
- 将业务拆分为多个实例加载，避免部分业务崩溃导致的整个App不可用
- 减少热更新带来的资源浪费，将RN包拆分为基础包、公共包及业务包，不常有更新的基础包和公共包排除在业务包之外，降低了业务包的大小，能够极大降低更新时的带宽占用