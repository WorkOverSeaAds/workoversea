---
title: React Native 打包与拆包
date: 2023-10-28 13:51:10
categories:
  - Front end
  - RN
tags:
  - RN
  - React Native
  - React
  - JavascriptCore
  - JSI
description: React Native 打包与拆包
---

## 1. 打包

react-native-cli提供了打包命令，我们可以配合metro.config.js来配置打包参数。

安装react-native-cli之后，在rn项目根目录下执行 react-native bundle -h 得到如下帮助信息。

```shell
Options:
  --entry-file <path>                Path to the root JS file, either absolute or relative to JS root
  --platform [string]                Either "ios" or "android" (default: "ios")
  --transformer [string]             Specify a custom transformer to be used
  --dev [boolean]                    If false, warnings are disabled and the bundle is minified (default: true)
  --minify [boolean]                 Allows overriding whether bundle is minified. This defaults to false if dev is true, and true if dev is false. Disabling minification can be useful for speeding up production builds for testing purposes.
  --bundle-output <string>           File name where to store the resulting bundle, ex. /tmp/groups.bundle
  --bundle-encoding [string]         Encoding the bundle should be written in (https://nodejs.org/api/buffer.html#buffer_buffer). (default: "utf8")
  --max-workers [number]             Specifies the maximum number of workers the worker-pool will spawn for transforming files. This defaults to the number of the cores available on your machine.
  --sourcemap-output [string]        File name where to store the sourcemap file for resulting bundle, ex. /tmp/groups.map
  --sourcemap-sources-root [string]  Path to make sourcemap's sources entries relative to, ex. /root/dir
  --sourcemap-use-absolute-path      Report SourceMapURL using its full path
  --assets-dest [string]             Directory name where to store assets referenced in the bundle
  --reset-cache                      Removes cached files
  --read-global-cache                Try to fetch transformed JS code from the global cache, if configured.
  --config [string]                  Path to the CLI configuration file
```

我们在打包时需要用到的几个参数有：

--entry-file：入口文件，一般使用根目录下的index.js作为入口文件即可。

--platform: 打包的平台，ios或android。

--dev: 是否为debug模式，默认是debug模式，因此需要将其设置为false。

--minify: 是否压缩代码，当dev是true的时候，minify默认为false。 反之当dev为false的时候minify默认为true。

--bundle-output: 打包后的jsbundle存储路径。

--assets-dest: 打包后资源存储路径，该路径会生成jsbundle中资源文件引用的相对路径，配置时需注意，尽量和jsbundle在同级目录下。

--sourcemap-output: 输出sourcemap的路径，不添加不会输出sourcemap。

--config: matroconfig配置文件位置，用来配置打包信息，可以做一些拆包配置。

## 2. 拆包

默认情况下， 我们直接通过如下命令就可以打出一个完整的RN包。

```shell
react-native bundle --entry-file ./index.js --platform ios --dev false --bundle-output ./build/ios/main.jsbundle --assets-dest ./build/ios/assets
```

但对于一个大型的项目，很多业务都是按照产品线或其他场景来划分的业务组来做的， 并不适合整个应用打出一个完整的包。我们可以采用类似于涂鸦智能这种，每个面板创建一个RCTRootView并由一个单独的RCTBridge维护的形式，优点就是不需要拆包，每个面板业务环境独立，不存在变量污染的情况。缺点就是，每个业务包都很大，下载比较消耗资源，每个面板都创建一套独立的js运行环境，不适合过多面板同时运行。

我们也可以采用拆包加载的形式，将每个面板业务拆分为三个部分。

1. RN框架部分，此部分为RN框架最核心的代码，需要在面板启动前加载完成保证后续的加载逻辑正常。此部分一般不会有变更，可以跟着App版本进行发布迭代。

2. 公共库部分，比如一些第三方依赖，tuya-panel-kit等。此部分可以单独发布热更新版本，用于修复公共库中的bug。

3. 业务部分，此部分为业务代码，需要根据业务需求进行变更，同时也可能添加一些非公共的第三方库依赖。

为了简单一些，我们将1和2合并为一个模块，在此我们称其为基础包。 3作为业务部分称为业务包。

首先我们为基础包生成一个配置文件，指定哪些文件需要打包进基础包，以及打包过程中每个模块的id如何定义。

```js
const pathSep = require('path').sep;
function createModuleIdFactory() { // 获取每个模块的id如何定义
  const projectRootPath = __dirname;//获取命令行执行的目录，__dirname是nodejs提供的变量
  return path => {
    let name = '';
    if(path.indexOf('node_modules'+pathSep+'react-native'+pathSep+'Libraries'+pathSep)>0){
      name = path.substr(path.lastIndexOf(pathSep)+1);//这里是去除路径中的'node_modules/react-native/Libraries/‘之前（包括）的字符串，可以减少包大小，可有可无
    }else if(path.indexOf(projectRootPath)==0){
      name = path.substr(projectRootPath.length+1);//这里是取相对路径，不这么弄的话就会打出_user_smallnew_works_....这么长的路径，还会把计算机名打进去
    }
    return name; // 这里我们使用文件的相对路径作为每个模块的moduleId
  };
}
function processModuleFilter(module){// 过滤该module是否需要打包
  if (
         module.path.indexOf('node_modules') > -1 // node_modules下被引用到的文件需要打包
  ){
    return true;
  }
  return false;
}
function getRunModuleStatement(entryFilePath){
  return `__r('indexCore$js')`// 配置最后需要require执行的入口文件
}
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: false,
      },
    }),
  },
  serializer: {
    createModuleIdFactory:createModuleIdFactory,
    processModuleFilter:processModuleFilter,
    getRunModuleStatement:getRunModuleStatement
  }
};

```

配置完成后， 我们使用该配置进行打包。

```shell
react-native bundle --config ./core.config.js  --entry-file ./indexCore.js --platform ${PLATFORM} --dev false --bundle-output ${OUTPUT_PATH}/${BIZ_FOLDER}.${PLATFORM}.js  --assets-dest ${OUTPUT_PATH}
```

打出RN包后， 我们查看内容与直接打包时的RN包有何差别。

```js
__d(function(g,r,i,a,m,e,d){'use strict';m.exports=r(d[0])},3,[4]);
__d(function(g,r,i,a,m,e,d){'use strict';m.exports=r(d[0])},"node_modules/react/index.js",["node_modules/react/cjs/react.production.min.js"]);
```

此时再看打包后的代码，原来数字定义的moduleId变为文件相对路径定义的moduleId之后，是不是变的更加清晰好理解了呢。

同样，我们通过修改processModuleFilter中的过滤规则，来打出来多个业务包。打包拆包部分就实现完成了。
