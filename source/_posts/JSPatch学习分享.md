---
title: JSPatch学习分享
date: 2016-12-23 10:38:08
tags: iOS
categories: [猿猿养成记]
---

## App热更新技术——JSPatch学习分享



 _如果不清楚本文的主角 `JSPatch`是什么请看我博客中的`JSPatch`学习笔记： [这里](/2016/12/19/JSPatch%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E4%B8%80/index.html) 和 [这里](/2016/12/19/JSPatch%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E4%BA%8C/index.html)_

### 背景


iOS Developer Program License Agreement里3.3.2提到不可动态下发可执行代码，但通过苹果JavaScriptCore.framework或WebKit执行的代码除外，JS正是通过JavaScriptCore.framework执行的。



### 基础原理



JSPatch 能做到通过 JS 调用和改写 OC 方法最根本的原因是 Objective-C 是动态语言，OC 上所有方法的调用/类的生成都通过 Objective-C Runtime 在运行时进行，我们可以通过类名/方法名反射得到相应的类和方法：

<!-- more -->

``` objectivec
Class class = NSClassFromString("UIViewController");
id viewController = [[class alloc] init];
SEL selector = NSSelectorFromString("viewDidLoad");
[viewController performSelector:selector];
```
也可以替换某个类的方法为新的实现：

``` objectivec
static void newViewDidLoad(id slf, SEL sel) {}
class_replaceMethod(class, selector, newViewDidLoad, @"");
```

还可以新注册一个类，为类添加方法：

``` objectivec
Class cls = objc_allocateClassPair(superCls, "JPObject", 0);
objc_registerClassPair(cls);
class_addMethod(cls, selector, implement, typedesc);
```
理论上可以在运行时通过类名/方法名调用到任何 OC 方法，替换任何类的实现以及新增任意类。JSPatch 的基本原理就是：JS 传递字符串给 OC，OC 通过 Runtime 接口调用和替换 OC 方法。

[JSPatch实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3)



### Demo展示



[demo地址](https://github.com/HuyangJake/JSPatchTestDemo)

疑问：热修复都需要重启App后才能生效吗？

demo 中对以上疑问有实现，具体的原理理解下一节  `方法替换`

对demo代码的具体解释请看 [这里](http://jake.gift/2016/12/19/JSPatch%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-%E4%BA%8C/)


### 方法替换



OC上，每个类底层都是这样一个结构体：

``` objectivec
struct objc_class {  
     Class isa;  
     Class super_class;    
     const char *name;   
     long version;  
     long info;   
     long instance_size;  
     struct objc_ivar_list *ivars;  
     struct objc_method_list **methodLists;  /*方法链表*/  
     struct objc_cache *cache;  
     struct objc_protocol_list *protocols;     
 }
```

其中 methodList 方法链表里存储的是 Method 类型：

``` objectivec
typedef struct objc_method *Method;
typedef struct objc_ method {
  SEL method_name;
  char *method_types;
  IMP method_imp;
};
```

Method 保存了一个方法的全部信息，包括 SEL 方法名，type 各参数和返回值类型，IMP 该方法具体实现的函数指针。

通过 Selector 调用方法时，会从 methodList 链表里找到对应Method进行调用，这个 methodList 上的的元素是可以动态替换的，可以把某个 Selector 对应的函数指针IMP替换成新的，也可以拿到已有的某个 Selector 对应的函数指针IMP，让另一个 Selector 跟它对应，Runtime 提供了一些接口做这些事



### JSPatch脚本文件下发加密


接入 JSPatch 时做 RSA 非对称加密传输
![](http://upload-images.jianshu.io/upload_images/611240-14723080a9823ced.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

开发者自己在 APP 接入 JSPatch，若开发者没有针对传输的 JSPatch 脚本加密。攻击者就通过网络传输的中间人攻击手段下发恶意脚本到用户APP



### 动态更新方案对比：



### JSPatch vs React Native


1. 学习成本

	 * React Native 是从 web 前端开发框架 React 延伸出来的解决方案，主要解决的问题是web页面在移动端性能低的问题。若使用React Native，就意味着iOS开发者需要学习web前端的一整套开发技能
	 
	 * JSPatch 是从终端开发出发的一种方案，JSPatch 写出来的代码风格与 OC 原生开发一致，加上一点 JS 语法的了解，就可以使用
2. 接入成本
	* React Native 需要搭建一套开发环境，有很多依赖的库。React Native 是比较大的框架，据统计目前核心代码里 OC 和 JS 代码加起来有4w行，接入后安装包体积增大 1.8M 左右
	
	* JSPatch 是微型框架，只有 3 个文件 2k 行代码，接入后增大 100K 左右
3. 开发效率
	* React Native 用近似 HTML+CSS 去绘制 UI，这方面开发效率相对 JSPatch 会高些，另外React Native 在开发效率上的另一个优势是支持跨平台，React Native 本意是复用逻辑层代码，UI 层根据不同平台写不同的代码
	
	* JSPatch 也可以借助 iOS 一些成熟的库去提高效率，例如使用 Massory。（尝试了其实也是相当吃力）
	
4. 热更新能力
	* React Native 在热更新时无法使用事先没有做过桥接的原生组件
	
	*  JSPatch 可以调用到任意已在项目里的组件，以及任意原生 framework 接口

方案对比表格：

||学习成本|接入成本|热更新能力|开发效率|性能体验|
|:---:|:-:|:-:|:-:|:-:|:-:|
|JSPatch|低|低|高|中，不跨平台|高|
|React Native|高|高|中|高，跨平台|高|

### JSPatch vs Wax

![](http://upload-images.jianshu.io/upload_images/611240-3d1af75ebfe7de01.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 对JSPatch的思考



1. 进行热更新后下一个版本需要将JS代码修改为原生代码，不能停留超过一个版本。

2. 编写JS脚本文件，调用OC原生的方法（方法名长得可怕）没有代码补全提示和高亮，显得非常吃力。

3. 当要进行热修复的方法是一个代码量很大的方法，需要用JS重写这个方法，这会很痛苦。




### 新方案 : DynamicCocoa



__优势所在：__

* 使用原生技术栈：使用者完全不用接触到 JS 或任何中间代码，保持原生的 Objective-C 开发、调试方式不变

* 无需重写已有代码：已有 native 模块能很方便的变成动态化插件

* 语法支持完备性高：支持绝大多数日常开发中用到的语法，不用担心这不支持那不支持

* 支持 HotPatch：改完 bug 后直接从源码打出 patch，一站式解决动态化和热修复需求

* 资源的支持,动态 bundle 支持：
	* xib 和 storyboard
	* xcassets
	* 不放在 xcassets 里的图片资源
	* 其他资源文件


[DynamicCocoa：滴滴 iOS 动态化方案的诞生与起航](http://mp.weixin.qq.com/s/qRW_akbU3TSd0SxpF3iQmQ)

