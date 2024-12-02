---
title: React Native 知识点
date: 2023-10-27 11:21:43
categories:
  - 前端
  - RN
tags:
  - RN
  - React Native
  - React
  - JavascriptCore
  - JSI
  - RN 原理
  - RN 加载流程
description: 对 React Native 基本概念及原理的介绍
---

## 1. React Native 基本概念

### 1.1 JavascriptCore(JSC)

要保证 RN 代码运行，首先要有一套 js 代码的运行环境，这套运行环境就是 JavascriptCore，但在 chrome 中进行调试的时候，由于 JS 代码都是在 chrome 的 V8 引擎中执行的，导致部分代码在 debug 模式下和非 debug 模式下存在差异。

比如：

1. ios 中部分日期函数未实现，比如在 IOS 中无法将 2021-08-26 日期转换成 Date 对象，而是需要将"-"替换为"/"再做转换，但在 chrome 中调试时却不会有这个问题。
2. 安卓在 debug 情况下，若有非法空白字符未处于 Text 标签下时，并不会报错，但在非 debug 环境下却会报红屏。因此开发时需要了解这些差异，避免出现此类情况。

关于 JavascriptCore 的详细介绍，可参照[官方文档](https://trac.webkit.org/wiki/JavaScriptCore)，里面详细介绍了 JSC 的组成部分及各部分的作用。

### 1.2 JSI(Javascript Interface)

JSI 是一个轻量级 c++库，通过 jsi，可以实现 js 直接对 c++层对象及方法进行调用。它是一个可运行于多种 js 引擎的中间适配层，有了 jsi，使得 RN 框架不仅可以基于 JSC 运行，还可以使用 V8 或 hermes 引擎。jsi 是 2018 年 facebook 对 RN 框架进行重构时引入的，引入后 RN 的架构也发生了较大的变化，性能得到了较大的提升。

### 1.3 jsbundle

有了 JS 运行时，还需要将可执行的代码加载到 App 中，jsbundle 就是我们需要的 JS 代码。

业务开发完成后， 我们会通过 react-native-cli 提供的打包命令进行打包，打包脚本会将我们开发的代码进行压编码，生成压缩后的 bundle 包， 如我们每个面板程序中打包后都会有一个 main.bundle 的 js 包， 而 js 中依赖的资源会根据相应的路径复制到对应的文件夹中。

#### a. 环境变量及方法定义

jsbundle 的第一行定义了运行时环境变量，用于表明所运行的 node 环境处于生产环境， 以及记录脚本启动的时间。

```JS
var __DEV__=false,__BUNDLE_START_TIME__=this.nativePerformanceNow?nativePerformanceNow():Date.now(),process=this.process||{};process.env=process.env||{};process.env.NODE_ENV="production";
```

第二到 10 行定义了全局方法，如**d、**c、\_\_r、setGlobalHandler、reportFatalError 等方法， 为 RN 环境启动的基本方法。

#### b. ReactNative 框架及业务代码定义

在第 11 行开始进入 React Native 框架、第三方库以及个人代码定义部分，该部分通过第二行定义的\_\_d 方法，对代码中的方法及变量进行定义。

\_\_d 方法接受 3 个参数：

第一个参数表示该模块（一般为一个文件中通过 export default 导出的部分）的定义。即我们或第三方开发人员写的某个代码文件中的代码逻辑。

第二个参数表示该模块的 moduleId，在其他模块对该模块进行引用时，需要通过该 id 来引用。 该值可以为数字，也可以为字符串，但要保证每个模块的 id 都是唯一的。 打包系统默认按照数字的递增的形式来定义该 id。

第三个参数表示该模块对其他模块的依赖，是一个数组，数组中的每个数字表示一个依赖的模块。

```JS
__d(function(g,r,i,a,m,e,d){var t=r(d[0]),n=t(r(d[1]));t(r(d[2]));r(d[3]);var l=r(d[4]),o=t(r(d[5])),s=r(d[6]),u=t(r(d[7])),p=t(r(d[8]));for(var f in l.TextInput.defaultProps=(0,n.default)({},l.TextInput.defaultProps,{allowFontScaling:!1}),l.Text.defaultProps=(0,n.default)({},l.Text.defaultProps,{allowFontScaling:!1}),console.disableYellowBox=!0,l.UIManager)l.UIManager.hasOwnProperty(f)&&l.UIManager[f]&&l.UIManager[f].directEventTypes&&(l.UIManager[f].directEventTypes.onGestureHandlerEvent={registrationName:"onGestureHandlerEvent"},l.UIManager[f].directEventTypes.onGestureHandlerStateChange={registrationName:"onGestureHandlerStateChange"});r(d[9]),r(d[10]),r(d[11]),g.userStore=(0,s.createStore)(u.default,(0,s.applyMiddleware)(p.default)),o.default.hide()},0,[1,2,3,6,18,416,417,420,422,423,656,727]);
__d(function(g,r,i,a,m,e,d){m.exports=function(n){return n&&n.__esModule?n:{default:n}}},1,[]);
__d(function(g,r,i,a,m,e,d){function t(){return m.exports=t=Object.assign||function(t){for(var n=1;n<arguments.length;n++){var o=arguments[n];for(var p in o)Object.prototype.hasOwnProperty.call(o,p)&&(t[p]=o[p])}return t},t.apply(this,arguments)}m.exports=t},2,[]);
__d(function(g,r,i,a,m,e,d){'use strict';m.exports=r(d[0])},3,[4]);
```

#### c. 引用与启动入口

只有定义了，还不足以使我们的 RN 应用运行起来， 如果要运行，还需要将我们的入口模块引用起来，于是此处就使用到了\_\_r 的方法。

\_\_r 方法接受一个参数，该参数为要引用的模块 id，若该模块没有被初始化，则尝试加载并初始化该模块，若未找到该模块，则会抛出 "Requiring unkonwn module ‘xxx’"的错误。

```JS
__r(104);
__r(0)
```

### 1.4 RCTBridge/ReactBridge

JS代码和JS运行时都准备好了，如何才能将这些代码运行起来呢？

做过或了解过RN开发的同学都知道，在RN开发中有一个Bridge的概念十分重要， 它在js端与原生端起到桥接的作用，是js与原生通讯和交互的基础，在整个RN生命周期中，它主要承担了如下工作。

a. 创建 RN 运行时

b. 执行 js 代码，将 jsbundle 加载并执行

c. 维护 js 与原生的双端通讯

d. 维护导出方法表及映射关系

### 1.5 RCTRootView/RCTShadowView

JS代码、运行时、JS执行对象都准备好了，我们还需要一个视图容器将React里画的界面渲染出来。

此时就需要RCTRootView出场了，RCTRootView作为RN的根容器，起到承载所有子视图的功能，但React的界面并非直接加载到RCTRootView上的， 它还有一层子视图RCTRootContentView，它才是直接承载视图的对象。

那RCTShadowView又是什么呢？ RCTShadowView是对RCT 视图树的镜像，类似于React中的虚拟Dom，负责维护每个视图实例的状态，js端发生变更时，首先由RCTShadowView收集并计算变化值，数据处理完成后，再将值同步给其对应的视图进行更新。

### 1.6 RCTUIManager

视图容器也有了，那界面里这么多视图，这么多组件的实例，由谁来管理呢？

UIManager承担了管理原生视图以及传递实图事件的责任，由原生端导致的视图实例都由UIManager进行管理，在创建视图时为每个视图实例添加唯一的tag作为key值，这样在js端需要操作视图时，只需要将tag和参数传递给UIManager，就可以定位到具体的视图实例。

### 1.7 RCTBridgeModule

js端和原生端如何进行通讯呢，我们又该如何去定义自己的方法让js进行调用呢？RN框架提供了一个协议RCTBridgeModule，实现这个协议就可以实现两端的通讯。

RCTBridge和RCTUIManager都实现了RCTBridgeModule协议，RN环境启动时会扫描所有实现该协议的类，生成一张表格，原生端和JS端都会保存这一张表格，它是一张映射表，有了这张映射表就可以让两端在调用方法时能够精准地找到对应的实现。

### 1.8 MessageQueue

有了通讯的桥梁还不足以支撑异步操作，若全部通讯及UI渲染都通过同步执行，那性能将会有很大的瓶颈。因此需要引入MessageQueue来作为通讯的池子，所有的通讯与交互事件都抛入池中，再通过规则去对池子进行读取刷新。MessageQueue就主要承担起异步事件交互通知的任务。这也是为什么调用原生方法时，原生端并不会立即生效的原因，比如锁定横屏操作时，在调用旋转为横屏的方法后，需要做一个延迟操作再将旋转锁定。

```JS
RCTOrientationManager.lockOrientation('landscape-left');
setTimeout(() => {
    RCTOrientationManager.shouldAutorotate(false);
}, 200);
```

## 2. React Native 运行原理

React Native 主要分为两部分： 一是React，即JSX层实现的视图及业务逻辑。二是Native，即原生端拿js层实现的逻辑进行的界面渲染。而这两部分之所以能够实现互通，依赖的是jsc（javascriptCore）,jsc 能够执行js代码，并通过解析将js代码中实现的视图映射为对应的原生组件， 按需执行js逻辑代码。正是由于jsc的存在，才使得我们可以通过js代码来写原生应用。 但仅靠jsc也无法正常运行起一个RN的应用，其中还依赖了一些其他实现。

### 2.1 RN整体架构

在jsi小节我们提到了RN的架构因jsi的引入发生了较大的变化。

#### 2.1.1 现版本架构

我们可以按逻辑所在线程来分为JS线程、UI线程（主线程）、Shadow线程三部分。其中Shadow线程主要负责上面提到的ShadowView更新的计算，这部分操作由c++层的yoga框架完成， 计算完成后再将数据交由主线程进行真实视图的刷新。

当React（js层）需要更新界面或调用原生端接口时，需要通过bridge将调用参数转换成JSON字符串，将json数据通过bridge传递到原生层，原生层通过解析json数据，找到对应的方法或视图执行操作，无法实现js层与原生层直接调用，同时在数据通讯时三个线程中的数据无法共享，只能各自保存一份，各自维护，三个线程中的通讯也只能通过异步调用。

其中NativeModules需要在启动的时候全部加载，并在原生端和js端务维护一个对象来保证在js端调用时能够正确地找到对应的方法，这部分操作在启动过程会占用较多的资源，且会加载大量使用不到的资源。

目前我们使用的0.59.10版本的RN框架，虽然已经引入的jsi架构，但实际上并没有在通讯中使用到，通讯依然是通过bridge实现的。

<image src="../assets/rn-1.png" />

#### 2.1.2 新版本架构

对比现版本架构，新版本引入了jsi和fabric的概念，jsi实现了js层和原生层的相互调用，fabric将替代UIManager，包含了renderer和shadow thread，而引入jsi之后，实现了多个线程之前的数据共享，不再需要使用json格式的数据相互传递，并保留副本。

<image src="../assets/rn-2.png" />

新版本的架构并依然使用三个线程进行并行处理，但三条线程都可以访问js线程中的数据，NativeModules引入了TurboModules技术也，再是在启动时全部加载，而是在使用时按需加载。同时新架构还引入了CodeGen的技术，能够根据types的定义，自动生成TurboModules原生代码，自动兼容线程之间的相互通讯。

新版本架构目前还未发布，但已在facebook内部应用中进行了迭代使用，预计下半年会有大版本更新发布。参照：[https://reactnative.dev/blog/2021/08/19/h2-2021](https://reactnative.dev/blog/2021/08/19/h2-2021)

### 2.2 新版本架构

了解了RN的基本架构，我们还需要了解一下当前架构的RN架构的启动流程是怎样的。其整体逻辑可简化为如下流程：

<image src="../assets/rn-3.png" />

#### 2.2.1 创建RCTRootView

上面提到RCTRootView是RN界面的原生容器，一般对于未拆包的RN应用来说， 在App启动时即可创建出RCTRootView作为App的根视图。

我们一般使用-(instancetype)initWithBundleURL:moduleName:initialProperties:launchOptions:方法来创建RCTRootView。传入四个参数分别是jsbundle的路径url；要启动的应用的名称（JS中通过AppRegistry.registerComponent注册的根组件名称）；初始化参数（会作为启动参数传入根组件的props）；App启动参数（一般不需要关心）；使用该方法创建RCTRootView时会同时创建出RCTBridge来维护整个RN应用的生命周期。

```js
   [[RCTRootView alloc] initWithBundleURL:[NSURL fileURLWithPath:panelPath] moduleName:@"Demo" initialProperties:initialProps launchOptions:launchOptions];
```

#### 2.2.2 C创建RCTBridge

RCTBridge是维护RN生命周期的重要对象，我们也可以在RCTBridge对你中提取需要的参数，需要的方法等。因此我们一般会更多地采用先创建一个RCTBridge再通过RCTBridge来创建RCTRootView的形式来初始化RN应用。

```objective-c
  RCTBridge *bridge = [[RCTBridge alloc] initWithDelegate:self launchOptions:launchOptions];
//或
  RCTBridge *bridge = [[RCTBridge alloc] initWithBundleURL:[[NSBundle mainBundle]	URLForResource:@"main" withExtension:@"jsbundle"] moduleProvider:nil 	launchOptions:launchOptions];
  RCTRootView *rootView = [[RCTRootView alloc] initWithBridge:bridge
                                                   moduleName:@"Demo"

                                            initialProperties:nil];
- (NSURL *)sourceURLForBridge:(RCTBridge *)bridge
{
#if DEBUG
  return [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index" fallbackResource:nil];
#else
  return [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
#endif
}
```

其中moduleProvider可以去配置该bridge可以访问到哪些NativeModules，在拆包应用控制权限时可以用到。

#### 2.2.3 RCTCxxBridge

RCTBridge初始化时会将创建时设置的bundleUrl保存，并创建一个RCTCxxBridge的实例batchedBridge去初始化RN的环境。 

```objective-c
  self.batchedBridge = [[bridgeClass alloc] initWithParentBridge:self];
  [self.batchedBridge start];

```

#### 2.2.4 加载RCTBridgeModule

batchedBridge启动首先发送将要加载的通知，之后创建js线程进行线程初始化，随后注册NativeModules。

```objective-c
 RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways, @"-[RCTCxxBridge start]", nil);


  [[NSNotificationCenter defaultCenter] postNotificationName:RCTJavaScriptWillStartLoadingNotification
                                                      object:_parentBridge
                                                    userInfo:@{@"bridge" : self}];


  // Set up the JS thread early
  _jsThread = [[NSThread alloc] initWithTarget:[self class] selector:@selector(runRunLoop) object:nil];
  _jsThread.name = RCTJSThreadName;
  _jsThread.qualityOfService = NSOperationQualityOfServiceUserInteractive;
#if RCT_DEBUG
  _jsThread.stackSize *= 2;
#endif
  [_jsThread start];
...



  [self registerExtraModules];
  // Initialize all native modules that cannot be loaded lazily
  (void)[self _initializeModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO];
  [self registerExtraLazyModules];

...
  dispatch_group_enter(prepareBridge);
  [self ensureOnJavaScriptThread:^{
    [weakSelf _initializeBridge:executorFactory];
    dispatch_group_leave(prepareBridge);
  }];


```

#### 2.2.5 执行JS代码

NativeModules加载完成之后开始读取jsbundle代码到内存中，读取完成后执行js代码。

```objective-c
  dispatch_group_enter(prepareBridge);
  __block NSData *sourceCode;
  [self
      loadSource:^(NSError *error, RCTSource *source) {
        if (error) {
          [weakSelf handleError:error];
        }


        sourceCode = source.data;
        dispatch_group_leave(prepareBridge);
      }
      onProgress:^(RCTLoadingProgress *progressData) {
#if (RCT_DEV | RCT_ENABLE_LOADING_VIEW) && __has_include(<React/RCTDevLoadingViewProtocol.h>)
        id<RCTDevLoadingViewProtocol> loadingView = [weakSelf moduleForName:@"DevLoadingView"
                                                      lazilyLoadIfNecessary:YES];
        [loadingView updateProgress:progressData];
#endif
      }];


  // Wait for both the modules and source code to have finished loading
  dispatch_group_notify(prepareBridge, dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0), ^{
    RCTCxxBridge *strongSelf = weakSelf;
    if (sourceCode && strongSelf.loading) {
      [strongSelf executeSourceCode:sourceCode sync:NO];
    }
  });
```

此时js代码已经执行到了内存中，根组件也已经注册，发送加载完毕的通知，此时RCTRootView可以创建RCTRootContentView了。

#### 2.2.6 runApplication

RCTRootView创建完成RCTRootContentView后立即调用AppRegistry.runApplication方法，开始加载RN逻辑进行渲染。此时开始走RN的逻辑流程。

```objective-c
  _contentView = [[RCTRootContentView alloc] initWithFrame:self.bounds
                                                    bridge:bridge
                                                  reactTag:self.reactTag
                                            sizeFlexiblity:_sizeFlexibility];
  [self runApplication:bridge];


- (void)runApplication:(RCTBridge *)bridge
{
  NSString *moduleName = _moduleName ?: @"";
  NSDictionary *appParameters = @{
    @"rootTag" : _contentView.reactTag,
    @"initialProps" : _appProperties ?: @{},
  };


  RCTLogInfo(@"Running application %@ (%@)", moduleName, appParameters);
  [bridge enqueueJSCall:@"AppRegistry" method:@"runApplication" args:@[ moduleName, appParameters ] completion:NULL];
}
```

#### 2.2.7 界面渲染上屏

RCTRootView调用runApplicaton之后，该方法调用会通过jscore以消息的形式将业务启动参数发送到js端的messageQueue(batchedBridge)中（上面提到过messageQueue是被动接收数据，主动定时刷新的形式。默认情况下messageQueue每5ms会进行一次flush操作，flush时发现新的消息会按照消息的参数进行逻辑的执行。），当执行到runApplication时会使用通过AppRegistry.registerComponent注册的组件进行界面的加载。 此时会通过UIManager根据Dom树中的组件及层级创建出dom配置，通过json的形式传递给原生端RCTUIManager， RCTUIManager再根据配置去创建或刷新Shadow，通过yoga计算出各组件的渲染参数（宽度、高度、位置等），将计算好的配置传递给对应的原生视图，如RCTView、RCTImage等组件，这些组件就会被按层级渲染到RCTRootContentView上，完成首屏的渲染。

<image src="../assets/rn-4.png" />

### 2.3 JSX与原生视图映射逻辑

上面提到js端uimanager需要将每个组件的配置传递给原生端的UIManager，UIManager再处理后续的渲染过程，那么两端的UIManager是如何统一配置的呢？

在1.6、1.7节中我们提到RCTUIManager遵循RCTBridgeModule协议，该协议允许注册模块及模块方法。2.2.4中又提到在RCTBridge初始化过程中，会去注册导出模块及方法，注册完成后js端即可拿到这些模块的方法的引用。

在模块注册时，其实是会在原生端和js端同时生成一份配置文件，remoteModuleConfig，在js端可以通过

\_\_fbBatchedBridgeConfig查看该变量。

js端在调用方法时，会根据这份配置文件，定位到原生端对应的组件进行相应的事件或配置的传递。

### 2.4 JS与原生通讯逻辑

JS可以通过导出模块或组件中的方法来触发原生端方法，这里使用触发是因为该操作是异步的，无法直接拿到原生端的返回值，只能通过回调或Promise的形式来处理返回值。此处也依赖了2.3中提到的模块注册时导出的配置文件。

<image src="../assets/rn-5.png" />

### 2.4.2 事件通知

事件通过机制就比较简单，而且双端都支持发送通知和注册监听，我们可以通过NativeAppEventEmitter, DeviceEventEmitter, NativeEventEmitter 来注册和发送通知。