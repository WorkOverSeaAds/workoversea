---
title: React Native 热更新及增量更新
date: 2023-10-28 13:11:00
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
  - bsdiff
  - Differential updating
description: React Native 热更新及增量更新
---

## 热更新

### 什么是热更新  

有了拆包和加载的逻辑，热更新其实就是下载新的业务包，替换掉旧的业务包。

### 如何保证热更新的版本可用

但是如果热更新下来的业务包一直崩溃怎么办呢？

方案一、 发布bugfix版本，修复热更新。前提：研发察觉到了线上崩溃。

方案二、 线上设置回滚，退回到上个版本。

方案三、 本地设置试运行机制，本地设置历史版本、试运行版本、正式版本三个版本目录，热更新版本首先添加到试运行版本中，首次运行成功后，再移动到正常版本目录，后续在正式版本目录中运行。若试运行连续崩溃到上限次数（如：2次），则回滚上一版本（在历史版本目录下找到最后的可运行版本），并记录本次热更新失败，再次尝试热更新，若同样达到上限次数依然崩溃，那么放弃本次热更新版本，直到有下一个热更新版本发布。

### 什么时机去更新

热更新有很多方案，比如启动App时更新全部业务包、定时轮询更新全部业务包、推送更新业务包、启动业务时检查并更新当前业务包等。

#### 启动App时更新全部业务包

业务逻辑最为简单，在App启动后首先请求热更新接口查询App中已有的业务包是否有新版本，若有新版本，则下载新版本等待运行，若业务在App启动后尚未启动过，那么使用新版本启动业务包，若已启动过则下次启动时使用新版本业务包。

优点： 启动后更新，业务逻辑处理简单，能保证多数场景下用户进入业务时使用的是最新版本的业务包。

缺点： 增加App启动耗时，进入业务时可能热更新尚未完成，使用的仍然是旧版本业务包。对于一些不经常杀进程退出应用的用户不友好。

#### 定时轮询更新全部业务包

对于热更新发布较频繁的应用，为保证业务版本更新率，定时轮询并下载热更新包也是一种较常见的方案。

优点： 能最大程度上保证热更新的触达率。

缺点： 对热更新接口考验较大，请求峰值可用性需要技术手段保证。定时请求存在较多的无效请求，浪费资源。

#### 推送更新业务包

对于多数场景来说，推送是一个比较好的手段，只在有版本时进行更新，不存在无效请求又能保证版本更新的实时性。

优点： 版本更新及时，不存在轮询方式带来的资源的浪费。

缺点： 安卓推送的兼容性较差，不能保证用户一定开启应用推送。

#### 启动业务时更新当前业务包

按需更新，只在用户启动业务时检测并下载新版本，更新完成后使用新版本进入业务。

优点： 保证用户使用的一定是最新版本。

缺点： 每次启动业务都要检查是否有新版本更新，存在资源浪费； 有新版本时，用户需要停在下载页面，下载完成后才能进入业务，用户体验较差。

## 增量更新

热更新功能虽然已经能够满足我们使用的需求了，但依然存在着资源的浪费。每次业务更新都要下载一个全量的业务包，但业务包的变更可能只有一行代码。那我们怎么再次去降低成本呢。

做过安卓开发的同学应该都知道，安卓有一项热修复的功能，就是打补丁包，将本次修改的内容与上次的进行对比，拆分出差异部分生成一个补丁包，旧版本的应用中下载到这个补丁包，将其合并到当前的版本中，就得到了新版本的App，多数应用商店都采用一这种技术。

我们git中也有diff方法，可以让我们查看两个文件的差异，其实增量更新和这git diff技术类似。就是遍历我们打出的业务包中的所有文件，找出有差异的文件，生成补丁包并压缩为对应版本的补丁包集合。 当我们App下载到补丁包后，将其与对应版本的业务包进行合并，合并完成后就得到了新版本的业务包。

增量更新和全量更新的差异到底有多大呢？

修改前的业务包内容：

<image src="../assets/rn-8.png" />

修改后的业务包内容：

<image src="../assets/rn-9.png" />

修改后的全量包大小

<image src="../assets/rn-10.png" />

修改后的增量包大小

<image src="../assets/rn-11.png" />

如果不做拆包的全量包又有多大呢

<image src="../assets/rn-12.png" />

这里的demo工程只有几个简单的页面，没有导入第三方依赖。想想如果导入了第三方依赖，再加上一些不经常变动的图片资源，那比例又会是怎样的呢。

## 如何加载新的RN包

下载下来RN包，直接使用新的RN包加载业务就能生效了吗？其实并不然。

我们都知道，我们在文件中通过import或require引入的依赖，最终都会被转换成reqiure的形式，而require是在metro中进行定义的。比如，我们在没有支持hot reload的RN框架下修改并保存了代码文件后，需要reload整个项目才会生效，这是因为require方法是有缓存策略的。

我们打开require.js文件，会发现如下的逻辑。

```js
  const module = modules[moduleIdReallyIsNumber];
  return module && module.isInitialized
    ? module.publicModule.exports
    : guardedLoadModule(moduleIdReallyIsNumber, module);

```

通过逻辑我们会发现，只有没有定义过对应moduleId的module时，require才会去读取并加载对应的代码，如果定义过，就会走缓存中的数据。因此，我们热更新下来的业务包，如果之前启动过一次，再次进入也不会使用新业务包里的逻辑。那我们怎么办呢？

通过上面的代码我们发现，模块代码会被缓存在modules变量中，那么我们退出业务时，把相关的module清理掉不就可以了？ 但什么时机去清理呢？

这里我们就用到了一个熟悉又陌生的Api， AppRegistry。 打开[AppRegistry文档](https://reactnative.cn/docs/appregistry)我们找到一个应用卸载的api，unmountApplicationComponentAtRootTag， 这个api会在应用退出卸载时调用，我们只要在这个里面做一些清理操作就可以了。 但清理哪些东西呢， 是不是需要我们记录一下，总不能把所有的modules清空吧。我们再看到runApplication方法，这个方法便是RCTRootView调用start时调用的js方法，它是应用启动的起点。我们可以根据入口参数在保存应用中依赖到了模块及名称。

```js
const initRunApplication = AppRegistry.runApplication;
const initUnmountApplicationComponentAtRootTag = AppRegistry.unmountApplicationComponentAtRootTag;
AppRegistry.runApplication = (appKey, appParameters) => {
  const { rootTag, initialProps: { bundleName, bundleUrl, entry } } = appParameters;
  definedBizModules[rootTag] = { entry, prefix: `src/pages/${bundleName}/` };
  new SourceTransformer(bundleUrl, bundleName)
  initRunApplication(appKey, appParameters);
}
AppRegistry.unmountApplicationComponentAtRootTag = (rootTag) => {
  initUnmountApplicationComponentAtRootTag(rootTag);
  const { entry, prefix } = definedBizModules[rootTag];
  console.log('unmount', entry)
  global.__destroyModules(entry, prefix)
}

```

为了保证应用正常运行，我们需要保留原方法，在我们的hook代码执行完成后，立即执行原方法。

global.__destroyModules是怎么定义的呢，这我们就需要修改一些require里的代码了。

```js
 const destroyModules = function (dModule, prefix) {
    if (typeof dModule === 'string') {
      delete modules[dModule];
      Object.keys(modules).forEach((item,i)=>{
          if(item.startsWith(prefix)){
            delete modules[item];
          }
      })
    }
  };
  global.__destroyModules = destroyModules;
```

我们将指定路径下的module全部清理，下次再进入的时候就可以正常加载了。

