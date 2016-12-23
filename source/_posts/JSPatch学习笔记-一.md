---
title: JSPatch学习笔记(一)
date: 2016-12-19 09:09:29
tags: iOS
---


### 什么是JSPatch
JSPatch 是一个开源项目([Github链接](https://github.com/bang590/JSPatch))，只需要在项目里引入极小的引擎文件，就可以使用 JavaScript 调用任何 Objective-C 的原生接口，替换任意 Objective-C 原生方法。目前主要用于下发 JS 脚本替换原生 Objective-C 代码，实时修复线上 bug。

理解下来类似于运行时的Method Swizzling，动态地将需要修复的bug代码替换成更新的代码。不同的是JSPatch使用JavaScript来进行热修复，可以在App上线的状态下替换App内的JS文件从而可以随心所欲修改和替换原有的方法，达到热修复的效果。

----
### JSPatch的使用
Github项目主页上介绍可以通过pod导入的方式加入项目，手动管理也仅仅是需要添加三个文件到项目既可使用。


>Installation
CocoaPods
[CocoaPods](http://cocoapods.org/) is a dependency manager for Objective-C, which automates and simplifies the process of using 3rd-party libraries like JSPatch in your projects. See the ["Getting Started"](https://guides.cocoapods.org/using/getting-started.html) guide for more information.

```
# Your Podfile
platform :ios, '6.0'
pod 'JSPatch'
```

[Manually](https://github.com/bang590/JSPatch#manually)
Copy `JSEngine.m` ` JSEngine.h` ` JSPatch.js`  in JSPatch/ to your project.

使用JS调用原生API的的JS语法在项目主页上也有[文档](https://github.com/bang590/JSPatch/wiki)。

其中最关键的就是这个API：

``` 
defineClass(classDeclaration, [properties,] instanceMethods, classMethods)
 @param classDeclaration: 字符串，类名/父类名和Protocol
 @param properties: 新增property，字符串数组，可省略
 @param instanceMethods: 要添加或覆盖的实例方法
 @param classMethods: 要添加或覆盖的类方法

```
例子：


``` javascript
defineClass('JPViewController', {
  handleBtn: function(sender) {
    //self.ORIGhandleBtn()
    var tableViewCtrl = JPTableViewController.alloc().init()
    self.navigationController().pushViewController_animated(tableViewCtrl, YES)
  }
})
```

代码作用是：替换`JPViewController` 类中的 `handleBtn:` 方法，并可以`self.ORIGhandleBtn()`选择执行原OC中的此方法，这就是修复（替换）bug代码。

----
在熟悉语法使用的过程中主要（暂时）遇到了如下几个问题（比较容易犯错）：
1. OC中的 `self.view` 和 `self.navigationController` 之类在这里都需要调用它们的getter方法来达到相同效果如`self.view()`
2. 在OC中写惯了的 `label.text = @"text"`之类都要使用setter方法`label.setText("text")`
3. 总结上面两条就是访问和赋值属性都要通过调用方法来实现
4. 在JS代码中添加属性，重写getter方法时，需要自己创建局部变量，不在有自动生成的_ivar可以使用

后话：方法名不再有补全提示千万不要写错了

正在继续学习……