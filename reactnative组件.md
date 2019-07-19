# reactnative_组件
本节介绍以下模块

RCTRootView,

RCTRootContentView,

RCTBridge,

RCTBatchedBridge,

RCTJavaScriptLoader,

RCTContextExecutor,

RCTModuleData,

RCTModuleMethod,

### RCTRootView

RCTRootView是React Native加载的地方,是万物之源。从这里开始，我们有了JS Engine, JS代码被加载进来，对应的原生模块也被加载进来，然后js loop开始运行。 js loop的驱动来源是Timer和Event Loop(用户事件). js loop跑起来以后应用就可以持续不停地跑下去了。

如果你要通过调试来理解RN底层原理，你也应该是从RCTRootView着手，顺藤摸瓜。

每个项目的AppDelegate.m的- (BOOL)application:didFinishLaunchingWithOptions:里面都可以看到RCTRootView的初始化代码，RCTRootView初始化完成以后，整个React Native运行环境就已经初始化好了，JS代码也加载完毕，所有React的绘制都会有这个RCTRootView来管理。

RCTRootView做的事情如下:

创建并且持有RCTBridge

加载JS Bundle并且初始化JS运行环境.

初始化JS运行环境的时候在App里面显示loadingView, 注意不是屏幕顶部的那个下拉悬浮进度提示条. RN第一次加载之后每次启动非常快，很少能意识到这个加载过程了。loadingView默认情况下为空, 也就是默认是没有效果的。loadingView可以被自定义，直接覆盖RCTRootView.loadingView就可以了.开发模式下RN app第一次启动因为需要完整打包整个js所以可以很明显看到加载的过程，加载第一次以后就看不到很明显的加载过程了，可以执行下面的命令来触发重新打包整个js来观察loadingView的效果 ` watchman watch-del-all && rm -rf node_modules/ && yarn install && yarn start – –reset-cache`, 然后杀掉app重启你就会看到一个很明显的进度提示.

JS运行环境准备好以后把加载视图用RCTRootContentView替换加载视图.

所有准备工作就绪以后调用AppRegistry.runApplication正式启动RN JS代码，从Root Component()开始UI绘制。

一个App可以有多个RCTRootView, 初始化的时候需要手动传输Bridge做为参数，全局可以有多个RCTRootView, 但是只能有一个Bridge.

如果你做过React Native和原生代码混编，你会发现混编就是把AppDelegate里面那段初始化RCTRootView的代码移动到需要混编的地方，然后把RCTRootView做为一个普通的subview来加载到原生的view里面去，非常简单。不过这地方也要注意处理好单Bridge实例的问题，同时，混编里面要注意RCTRootView如果销毁过早可能会引发JS回调奔溃的问题。

### RCTRootContentView

RCTRootContentView reactTag在默认情况下为1. 在Xcode view Hierarchy debugger 下可以看到，最顶层为RCTRootView, 里面嵌套的是RCTRootContentView, 从RCTRootContentView开始，每个View都有一个reactTag.

RCTRootView继承自UIView, RCTRootView主要负责初始化JS Environment和React代码，然后管理整个运行环境的生命周期。 RCTRootContentView继承自RCTView, RCTView继承自UIView, RCTView封装了React Component Node更新和渲染的逻辑， RCTRootContentView会管理所有react ui components. RCTRootContentView同时负责处理所有touch事件.

### RCTBridge

这是一个加载和初始化专用类，用于前期JS的初始化和原生代码的加载。

负责加载各个Bridge模块供JS调用

找到并注册所有实现了RCTBridgeModule protocol的类, 供JS后期使用. 

创建和持有 RCTBatchedBridge

### RCTBatchedBridge

如果RCTBridge是总裁, 那么RCTBatchedBridge就是副总裁。前者负责发号施令，后者负责实施落地。

负责Native和JS之间的相互调用(消息通信)

持有JSExecutor

实例化所有在RCTBridge里面注册了的native node_modules

创建JS运行环境, 注入native hooks 和modules, 执行 JS bundle script

管理JS run loop, 批量把所有JS到native的调用翻译成native invocations

批量管理原生代码到JS的调用，把这些调用翻译成JS消息发送给JS executor

### RCTJavaScriptLoader

这是实现远程代码加载的核心。热更新，开发环境代码加载，静态jsbundle加载都离不开这个工具。

从指定的地方(bundle, http server)加载 script bundle

把加载完成的脚本用string的形式返回

处理所有获取代码、打包代码时遇到的错误

### RCTContextExecutor

封装了基础的JS和原生代码互掉和管理逻辑，是JS引擎切换的基础。通过不同的RCTCOntextExecutor来适配不同的JS Engine，让我们的React JS可以在iOS、Android、chrome甚至是自定义的js engine里面执行。这也是为何我们能在chrome里面直接调试js代码的原因。

管理和执行所有N2J调用

### RCTModuleData

加载和管理所有和JS有交互的原生代码。把需要和JS交互的代码按照一定的规则自动封装成JS模块。

收集所有桥接模块的信息，供注入到JS运行环境

### RCTModuleMethod

记录所有原生代码的导出函数地址(JS里面是不能直接持有原生对象的)，同时生成对应的字符串映射到该函数地址。JS调用原生函数的时候会通过message的形式调用过来。

记录所有的原生代码的函数地址，并且生成对应的字符串映射到该地址

记录所有的block的地址并且映射到唯一的一个id

翻译所有J2N call，然后执行对应的native方法。

如果是原生方法的调用则直接通过方法名调用，MessageQueue会帮忙把Method翻译成MethodID, 然后转发消息给原生代码，传递函数签名和参数给原生MessageQueue, 最终给RCTModuleMethod解析调用最终的方法

如果JS调用的是一个回调block，MessageQueue会把回调对象转化成一个一次性的block id, 然后传递给RCTModuleMethod, 最终由RCTModuleMethod解析调用。基本上和方法调用一样，只不过生命周期会不一样，block是动态生成的，要及时销毁，要不然会导致内存泄漏。

注:

实际上是不存在原生MessageQueue对象模块的，JS的MessageQueue对应到原生层就是RCTModuleData & RCTModuleMethod的组合, MessageQueue的到原生层的调用先经过RCTModuleData和RCTModuleMethod翻译成原生代码调用，然后执行.

## BridgeFactory

## JSCExecutor

## MessageQueue

这是核心中的核心。整个react native对浏览器内核是未做任何定制的，完全依赖浏览器内核的标准接口在运作。它怎么实现UI的完全定制的呢？它实际上未使用浏览器内核的任何UI绘制功能，注意是未使用UI绘制功能。它利用javascript引擎强大的DOM操作管理能力来管理所有UI节点，每次刷新前把所有节点信息更新完毕以后再给yoga做排版，然后再调用原生组件来绘制。javascript是整个系统的核心语言。

我们可以把浏览器看成一个盒子，javascript引擎是盒子里面的总管，DOM是javascript引擎内置的，javascript和javascript引擎也是无缝链接的。react native是怎么跳出这个盒子去调用外部原生组件来绘制UI的呢？秘密就在MessageQueue。

javascript引擎对原生代码的调用都是通过一套固定的接口来实现，这套接口的主要作用就是记录原生接口的地址和对应的javascript的函数名称，然后在javascript调用该函数的时候把调用转发给原生接口

# React Native 初始化

React Native的初始化从RootView开始，默认在AppDelegate.m:- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions 里面会有RootViewd的初始化逻辑，调试的时候可以从这里入手。 

React Native的初始化分为几个步骤:

原生代码加载

JS Engine初始化(生成一个空的JS引擎)

JS基础设施初始化. 主要是require等基本模块的加载并替换JS默认的实现。自定义require, Warning window, Alert window, fetch等都是在这里进行的。基础设施初始化好以后就可以开始加载js代码了。

遍历加载所有要导出给JS用的原生模块和方法, 生成对应的JS模块信息，打包成json的格式给JS Engine, 准确地说是给MessageQueue. 这里需要提一下的是

这里的导出是没有对象的，只有方法和模块。JS不是一个标准的面向对象语言，刚从Java转JavaScript的同学都会在面向对象这个概念上栽跟头，这里特别提醒一下。

## 原生代码初始化

这里讨论的主要是RN相关的原生代码和用户自定义的RN模块的原生代码的加载和初始化。原生代码初始化主要分两步：

静态加载。iOS没有动态加载原生代码的接口，所有的代码都在编译的初期就已经编译为静态代码并且链接好，程序启动的时候所有的原生代码都会加载好。这是原生代码的静态加载，iOS里面没有动态加载原生代码的概念，这也是为何没有静态代码热更新的原因。

RN模块解析和注入JS。这是加载的第二步。在RootView初始化的时候会遍历所有被标记为RCTModule的原生模块，生成一个json格式的模块信息，里面包含模块名称和方法名称，然后注入到JS Engine, 由MessageQueue记录下来。原生代码在生成json模块信息的时候同时会在原生代码这边维护一个名称字典，用来把模块和方法的名称映射到原生代码的地址上去，用于JS调用原生代码的翻译。

接下来我们就一步一步详细讲解原生代码的初始化。

## Javascript环境初始化

RN的初始化是从RCRootView开始的，所有的绘制都会在这个RootView里面进行(Alert除外).

RootView做的第一件事情就是初始化一个空的JS Engine。 这个空的JS Engine里面包含一些最基础的模块和方法(fetch, require, alert等), 没有UI绘制模块。 RN的工作就是替换这些基础的模块和方法，然后把RN的UI绘制模块加载并注入到JS Engine.

JS Engine不直接管理UI的绘制。

所有的绘制由原生控制的UI事件和Timer触发

影响界面刷新的事件发生以后一部分直接由原生控件消化掉，直接更新原生控件。剩下的部分会通过Bridge派发给MessageQueue，然后在JS层进行业务逻辑的计算，再由React来进行Virtual Dom的管理和更新。Virtual Dom再通过MessageQueue发送重绘指令给对应的原生组件进行UI更新。

## NativeModules加载

在OC里面，所有NativeModules要加载进JS Engine都必须遵循一定的协议(protocol)。

模块(OC里面的类)需要声明为<RCTBridgeModule>, 然后在类里面还必须调用宏RCT_EXPORT_MODULE() 用来定义一个接口告诉JS当前模块叫什么名字。这个宏可以接受一个可选的参数，指定模块名，不指定的情况下就取类名。 

对应的JS模块在初始化的时候会调用原生类的[xxx new]方法.

模块声明为<RCTBridgeModule>后只是告诉Native Modules这有一个原生模块，是一个空的模块。要导出任何方法给JS使用都必须手动用宏RCT_EXPORT_METHOD来导出方法给JS用.

所有的原生模块都会注册到NativeModules这一个JS模块下面去，你如果想要让自己的模块成为一个顶级模块就必须再写一个JS文件封装一遍NativeModules里面的方法。

你如果想自己的方法导出就默认成为顶级方法，那么你需要一个手动去调用JSC的接口，这个在前面章节有讲解。 不建议这样做，因为这样你会失去跨JS引擎的便利性。

你可以导出常量到JS里面去, 模块初始化的时候会坚持用户是否有实现constantsToExport 方法, 接受一个常量词典。 

\- (NSDictionary *)constantsToExport

{

return @{ @"firstDayOfTheWeek": @"Monday" };// JS里面可以直接调用 ModuleName.firstDayOfTheWeek获取这个常量

}

常量只会在初始化的时候调用一次，动态修改该方法的返回值无效

所有标记为RCT_EXPORT_MODULE的模块都会在程序启动的时候自动注册好这些模块，主要是记录模块名和方法名。只是注册，不一定会初始化。

Native Modules导出宏具体使用方法见官方文档[Native Modules](https://facebook.github.io/react-native/docs/native-modules-ios.html)

## NativeModules懒加载

React Native的NativeModules是有延迟加载机制的。App初始化的时候

React Native JS接口兼容(Polyfills)

fetch替换

CommonJS Require

alert替换

console.warning替换

console.error替换

线程讲解

JS单线程和其背后的Native多线程

JS的异步

三大线程

消息通信

Native调用JS代码

一般JS调用

ReactNative里面的JS调用

Native实现Promise

原生对象管理

模块管理

UIManager

TagID

JS调用Native代码

JS调用NativeModule代码

JS回调Native代码

渲染

ReactJS渲染机制

ReactFiber/ReactStack

ReactNative渲染

热加载

JS Reload

JS代码打包

metro-bundler

JSX

babel

Websocket

YellowBox详解

RedBox详解

调试菜单

自定义一个原生控件

SplashScreen工作逻辑

## 三个线程

React Native有三个重要的线程:

Shadow queue. 布局引擎([yoga](https://facebook.github.io/yoga/))计算布局用的。

Main thread. 主线程。就是操作系统的UI线程。无论是iOS还是android，一个进程都只有一个UI线程，我们常说的主线程. React Native所有UI绘制也是由同一个UI线程来维护。

Javascript thread. javascript线程。 大家都知道javascript是单线程模型，event驱动的异步模型。React Native用了JS引擎，所以也必需有一个独立的js 线程. 所有JS和原生代码的交互都发生在这个线程里。死锁，异常也最容易发生在这个线程.

可以看到Shadow queue是queue而不是thread, 在iOS里面queue是thread之上的一层抽象,GCD里面的一个概念，创建queue的时候可以指定是并行的还是串行的。也就是说，一个queue可能对应多个thread。

As mentioned above, every module will have it’s own GCD Queue by default, unless it specifies the queue it wants to run on, by implementing the -methodQueue method or synthesizing the methodQueue property with a valid queue.



