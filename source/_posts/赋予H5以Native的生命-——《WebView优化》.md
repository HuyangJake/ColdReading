---
title: 赋予H5以Native的生命 ——《WebView优化》
date: 2018-05-02 18:07:40
tags: [iOS]
---

本文是从客户端为起点发散到WebView使用过程中的优化点。当前市场上的Native App中WebView使用的情况还是比较多的，像手机QQ，淘宝业务更替快速的产品，使用WebView动态渲染页面是必然的选择(或者说曾经)，然而遵从这个选择就必须承担着它带来的弊病，更确切地说应该是尽可能解决它的弊病。

## 前言

众所周知在移动端使用WebView给人最直观的感觉是慢。造成这个现象的原因是多层次的，不过主要可以归纳为两个方面<sup>[2]</sup> ：

* 页面启动时间：打开一个 H5 页面需要做一系列处理，会有一段白屏时间，体验糟糕。
* 响应流畅度：由于 webkit 的渲染机制，单线程，历史包袱等原因，页面刷新/交互的性能体验不如原生。

<!--more-->

在WebView的先天缺陷角度--响应流畅度 大厂给出解决方案： FB 的 React-Native（多称RN） 和 阿里的 Weex。 
RN和Weex的核心实现跟WebView并没有关系，它们实现的并不是Hybrid App，里面是使用JavaScript引擎执行JS调用原生的组件。

不过它们的理念有一定的差异性

| weex |react-native|
|:-:|:-:|
|Vue.js|	React|
|write once, run anywhere	|learn once, write anywhere

这都不是我们今天的主角，今日主场属于WebView，那么我们所面临的问题是： __H5页面启动时间__

## 流程分析

下图是一个H5页面展示过程中要经历的流程：
![H5页面加载流程](http://qiniu.huyangjie.cn/H5%E9%A1%B5%E9%9D%A2%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B.png)

简单的页面可能会没有JS请求数据这一个步骤，一般页面在 dom 渲染后能显示雏形，在这之前用户看到的都是白屏，等到下载渲染图片后整个页面才完整显示，打开页面优化就是要减少这个过程的耗时。

----

## 优化


### 前端
B/S结构模式上对web的优化已经有做得比较极致方法，本文就不再介绍了（作为客户端同学也没有什么经验）这里给一个参考传送门<sup>[5]</sup>：
[唯快不破：Web 应用的 13 个优化步骤](https://zhuanlan.zhihu.com/p/21417465?refer=no-backend)

比较重要也是优化效果比较显著的是能够熟悉[HTTP缓存协议](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn)使用，这个需要服务端配合一起优化。


### 客户端
客户端上的H5展示跟传统的web页面有所不同，有优势也更有劣势。相对于PC端的处理器，移动客户端上的CPU性能会有所差距。不过由于客户端的H5是通过WebView内嵌在App中，情况不会像传统web一样，所有的优化都受限在浏览器之下，在客户端我们可以拿到更多的权限，做更深的优化。

#### 缓存（主要针对iOS端，慎读！大片OC代码风格可能引起您的不适）

客户端可以拦截所有的网络请求，并自己实现缓存机制。iOS端的Cocoa框架UIWebView提供了客户端层面的缓存策略：

``` objectivec
NSURLRequestUseProtocolCachePolicy  //缓存策略定义在 web 协议实现中，用于请求特定的URL。是默认的URL缓存策略。
NSURLRequestReloadIgnoringLocalCacheData //从服务端获取数据，忽略本地缓存
NSURLRequestReloadIgnoringLocalAndRemoteCacheData //源文件注释中写到没有实现
NSURLRequestReloadIgnoringCacheData //被NSURLRequestReloadIgnoringLocalCacheData替换了
NSURLRequestReturnCacheDataElseLoad //已经存在的缓存数据用于请求返回，不管它的过期日期和已经存在了多久。如果没有请求对应的缓存数据，从数据源读取
NSURLRequestReturnCacheDataDontLoad //已经存在的缓存数据用于请求返回，不管它的过期日期和已经存在了多久。如果没有请求对应的缓存数据，不要去数据源读取，该请求被设置为失败，这种情况多用于离线模式
NSURLRequestReloadRevalidatingCacheData //源文件中写到没有实现
```

其中默认缓存策略（最通用）`NSURLRequestUseProtocolCachePolicy`的流程如下：
![](http://qiniu.huyangjie.cn/15252477856431.jpg)


其实我们自定义缓存策略，加上更多优化的点，比如自定义缓存的存储方式能够实现离线缓存，又能够实时更新，定义本地缓存失效时间等。

自定义流程的实现在iOS中是通过继承`NSURLProtocol`拦截处理所有的网络请求来实现的，这是我的一种实现方案：[YJURLProtocol](https://github.com/HuyangJake/YJURLProtocol)  当然GitHub上有很多实现方案，最好是根据自己的需求去自定义。

![](http://qiniu.huyangjie.cn/15252481937129.jpg)

----

#### WKWebView是否支持NSURLProtocol？

在使用`WKWebView`加载网页的时候，`NSURLProtocol`子类会不能拦截到请求，原因是`WKWebView`的请求是在单独的进程里，所以不会走`NSURLProtocol`。当然解决办法是有的，因为其实`WKWebView`是支持`NSURLProtocol`协议的，只是还不够完善，当前可以通过调用私有的API去完成这项任务（详细分析过程见[参考[6]](https://www.jianshu.com/p/55f5ac1ab817)）,以下是实现的关键代码：

``` objectivec
FOUNDATION_STATIC_INLINE Class ContextControllerClass() {
    static Class cls;
    if (!cls) {
        cls = [[[WKWebView new] valueForKey:@"browsingContextController"] class];
    }
    return cls;
}

FOUNDATION_STATIC_INLINE SEL RegisterSchemeSelector() {
    return NSSelectorFromString(@"registerSchemeForCustomProtocol:");
}

FOUNDATION_STATIC_INLINE SEL UnregisterSchemeSelector() {
    return NSSelectorFromString(@"unregisterSchemeForCustomProtocol:");
}

@implementation NSURLProtocol (WebKitSupport)

+ (void)wk_registerScheme:(NSString *)scheme {
    Class cls = ContextControllerClass();
    SEL sel = RegisterSchemeSelector();
    if ([(id)cls respondsToSelector:sel]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [(id)cls performSelector:sel withObject:scheme];
#pragma clang diagnostic pop
    }
}

+ (void)wk_unregisterScheme:(NSString *)scheme {
    Class cls = ContextControllerClass();
    SEL sel = UnregisterSchemeSelector();
    if ([(id)cls respondsToSelector:sel]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [(id)cls performSelector:sel withObject:scheme];
#pragma clang diagnostic pop
    }
}

@end
```

附上解决方案源码：[https://github.com/Yeatse/NSURLProtocol-WebKitSupport](https://github.com/Yeatse/NSURLProtocol-WebKitSupport)
其中会使用到私有的API，关于使用私有API肯定担心的是能不能通过Apple的审核。摘录一段前百度工程师sunnyxx的描述：

>关于私有API
大家会质疑说，这用到了 UIKit 的私有属性和私有 API，要是系统升级变了咋办？要是审核被拒了咋办？
首先，iOS 系统的 SDK 为了向下兼容，一般只会增加方法或者修改方法实现，不太可能直接删除一个共有方法，而私有方法的行为确实可能有变化，但系统 release 频率毕竟很低，每当新版本发布时 check 下原来的功能是否能 work 就好了，大可不必担心这么远，SDK 是死的人是活的。
不论是 kvc 还是 selector 反射，都是利用 objc runtime 完成的，而到了这一层，真的就没啥公有私有可言了。设想你就是开发 Apple 私有 API 检查工具的工程师，给你一个 ipa 的包，你会如何检查出其中有没有私有 API 呢？

>首先，这个检查一定是个静态检查，不可能是运行时检查，因为代码逻辑那么复杂，把程序跑起来看所有 objc_msgSend 中包不包括私有调用这件事太不现实了。
对 ipa 文件做静态检查的话肯定是去分析 Mach-O 可执行文件，因为这时很多源代码级别的信息已经丢失，经分析可以采取下面几种手段：

>是否 link 了私有 framework 或者公开 framework 中的私有符号，这可以防止开发者把私有 header 都 dump 出来供程序直接调用。
同上，使用`@selector(_private_sel)`加上`-performSelector:`的方式直接调用私有 API。
扫描所有符号，查看是否有继承自私有类，重载私有方法，方法名是否有重合。
扫描所有string，看字符串常量段是否出现和私有 API 对应的。
我觉得前三条被 catch 住的可能性最高，也最容易被检查出来。再来看我们用到用字符串的方法 kvc 和 反射 selector，应该属于最后一条，这时候就很难抉择了，拿 `handleNavigationTransition:` 来说，看上去人畜无害啊，我自己类里面的方法也完全可能命名出这个来，所以单单凭借字符串命中私有 API 判定，苹果很容易误伤一大票开发者。
综上，我觉得使用字符串的方式使用私有 API 是相对安全的


PS: iOS11之后可以通过[`WKURLSchemeHandler`](https://developer.apple.com/documentation/webkit/wkurlschemehandler)去完成对`WKWebView`的请求拦截,不需要再调用私有API解决上述问题了。

---

上述方案似乎已经完美解决缓存问题，但实际上还有很多问题：

* 没有预加载：第一次打开的体验很差，所有数据都要从网络请求。
* 缓存不可控：缓存的存取由系统 webview 控制，无法控制它的缓存逻辑，带来的问题包括： 
    * i. 清理逻辑不可控，缓存空间有限，可能缓存几张大图片后，重要的 HTML/JS/CSS 缓存就被清除了。 
    * ii.磁盘 IO 无法控制，无法从磁盘预加载数据到内存。
* 更新体验差：后台 HTML/JS/CSS 更新时全量下载，数据量大，弱网下载耗时长。
* 无法防劫持：若 HTML 页面被运营商或其他第三方劫持，将长时间缓存劫持的页面。

还有一个方案就是使用zip包存放HTML文件和资源文件，进行统一管理。

#### 离线H5 zip 包

以下是现任职蚂蚁金服的`bang`提供的一个比较完善的客户端离线包方案：

1. 后端使用构建工具把同一个业务模块相关的页面和资源打包成一个文件，同时对文件加密/签名。
2. 客户端根据配置表，在自定义时机去把离线包拉下来，做解压/解密/校验等工作。
3. 根据配置表，打开某个业务时转接到打开离线包的入口页面。
4. 拦截网络请求，对于离线包已经有的文件，直接读取离线包数据返回，否则走 HTTP 协议缓存逻辑。
5. 离线包更新时，根据版本号后台下发两个版本间的 diff 数据，客户端合并，增量更新。

我在项目中准备实践这个方案，当前只使用了一步，配置业务转接入口+本地zip包解密解压进行加载。 项目使用`SSZipArchive`对zip包进行解压，放到temporary目录加载资源。解压zip包和迁移文件的工作放在App启动之后的异步线程执行，不会影响App的启动速度。

``` objectivec
- (void)unZipFile {
    NSString *zipFile = [[NSBundle mainBundle] pathForResource:@"dist" ofType:@"zip"];
    NSString *destinationPath = [NSURL fileURLWithPath:NSTemporaryDirectory()].path;
    [SSZipArchive unzipFileAtPath:zipFile toDestination:destinationPath overwrite:YES password:@"******" error:nil];
}
```

关于离线包的增量更新方案的参考:
[实现前端资源增量式更新的一种思路](https://zhuanlan.zhihu.com/p/23218754)
[两种增量更新方案](http://blog.cnbang.net/tech/2258/)

这一块还有很多需要实践的点..

#### 预加载 webview

无论是 iOS 还是 Android，本地 webview 初始化都要不少时间，可以预先初始化好 webview。这里分两种预加载：

* 首次预加载：在一个进程内首次初始化 webview 与第二次初始化不同，首次会比第二次慢很多。原因预计是 webview 首次初始化后，即使 webview 已经释放，但一些多 webview 共用的全局服务或资源对象仍没有释放，第二次初始化时不需要再生成这些对象从而变快。我们可以在 APP 启动时预先初始化一个 webview 然后释放，这样等用户真正走到 H5 模块去加载 webview时就变快了。
* webview 池：可以用两个或多个 webview 重复使用，而不是每次打开 H5 都新建 webview。不过这种方式要解决页面跳转时清空上一个页面，另外若一个 H5 页面上 JS 出现内存泄漏，就影响到其他页面，在 APP 运行期间都无法释放了。

----

## 总结
WebView层面加载提高性能最大的优化方向还是缓存，预加载，在有限的资源和时间内更合理地调度资源。

## Reference 

- [1]. [Http缓存](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn)
- [2]. [移动 H5 首屏秒开优化方案探讨](https://blog.cnbang.net/tech/3477/)
- [3]. [WebView性能、体验分析与优化](https://tech.meituan.com/WebViewPerf.html)
- [4]. [70%以上业务由H5开发，手机QQ Hybrid 的架构如何优化演进？](https://mp.weixin.qq.com/s/evzDnTsHrAr2b9jcevwBzA)
- [5]. [唯快不破：Web 应用的 13 个优化步骤](https://zhuanlan.zhihu.com/p/21417465?refer=no-backend)
- [6]. [一个完美的半成品-WKWebView](https://www.jianshu.com/p/55f5ac1ab817)

