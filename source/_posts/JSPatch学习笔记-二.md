---
title: JSPatch学习笔记(二)
date: 2016-12-19 09:14:01
tags: [iOS] 
categories: [猿猿养成记, 学习笔记]
---

###  这次笔记中主要描述的有：
* JSPath原理理解（学习作者大牛博客）
* JSPatch使用的时机
* AppDelegate中更新JS文件后何时生效
* 其他时机手动更新JS文件的效果
* JS调用OC方法中的几个坑
* JS脚本文件的版本控制管理
* 更多思考



----
#### 1. JSPatch原理浅谈
JSPatch用iOS内置的JavaScriptCore.framework作为JS引擎，但没有用它JSExport的特性进行JS-OC函 数互调，而是通过Objective-C Runtime，从JS传递要调用的类名函数名到Objective-C，再使用NSInvocation动态调用对应的OC方法。

<!--more-->

详细原理介绍可见作者博客:[JSPatch 实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3)

另外JSPatch已经有商业化的平台[jspatch.com](http://jspatch.com/)，可以使用里面的SDK，通过这个平台上传的js脚本都存储在七牛云。

#### 2. JSPatch使用的时机
确切地说应该是在经过怎样的流程之后开始载入调用JS脚本。
在董铂然的博客:[JSPatch使用小记](http://www.cnblogs.com/dsxniubility/p/5080875.html)中有这样的一套方案：
![](http://upload-images.jianshu.io/upload_images/2474800-2bd018777a64f24e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/2474800-fe734288e695601a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个方案的特点：
1. 添加了上次请求的时间，避免多余的网络请求
2. 把更新js脚本的代码放在了`applicationDidBecomeActive:`方法中，避免程序在后台的时候也进行不必要的脚本更新检查。
3. 对js文件进行code校验，避免传输过程中被修改。实际使用中应对js脚本文件进行加密。（作者的博客中也建议用RSA等非对称加密对文件进行加密传输）
4. 连续崩溃次数的判断，能够做到程序自我选择性修复
 





#### 3. AppDelegate中更新JS文件后何时生效
当经过本文第二点中流程之后，本地载入了最新的js脚本文件，那么程序什么时候会使用这个脚本中的代码呢？
为此我写了[demo](https://github.com/HuyangJake/JSPatchTestDemo.git)测试：

AppDelegate中的代码

判断本地是否存在hotfix.js文件：

``` objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    NSString *docuPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
    NSString *hotfixPath = [docuPath stringByAppendingPathComponent:@"hotfix.js"];
    if ([[NSFileManager defaultManager] fileExistsAtPath:hotfixPath]) {
        [JPEngine startEngine];
        NSString *script = [NSString stringWithContentsOfFile:hotfixPath encoding:NSUTF8StringEncoding error:nil];
        [JPEngine evaluateScript:script];
    }
    
    return YES;
}
```

检测是否需要更新：

``` objectivec
- (void)applicationDidBecomeActive:(UIApplication *)application {
    // Restart any tasks that were paused (or not yet started) while the application was inactive. If the application was previously in the background, optionally refresh the user interface.
    
    [HotFixManager checkUpdateCompleteHandle:^(BOOL status, NSString *response, NSError *error) {
        dispatch_async(dispatch_get_main_queue(), ^{
            if (!status){
                if (!error) {
                    NSLog(@"没有更新");
                } else {
                    NSLog(@"%@", error.userInfo);
                }
                return ;
            }
            NSLog(@"Hotfix文件更新成功");
            [JPEngine startEngine];
            NSString *script = [NSString stringWithContentsOfFile:response encoding:NSUTF8StringEncoding error:nil];
            [JPEngine evaluateScript:script];
            
        });
    }];
   
}
```

* 首先第一点可以确定的是：
**判断本地是否存在js文件需要加载的代码不能够写在网络请求是否需要更新的回调中**
原因： 
首先看下我的hotfix.js文件中的代码

``` javascript
defineClass('JSPatchController', [/*新增的属性*/'updateLabel'], {//实例方法
            
            viewDidLoad: function() {
//            self.ORIGviewDidLoad()
            self.setTitle("测试JS1")
            self.view().addSubview(self.getUpdateLabel())
            },
            
            //实现Label的getter方法
            getUpdateLabel: function() {
            var _updateLabel = self.updateLabel()
            if (!_updateLabel) {
                _updateLabel = require('UILabel').alloc().init()
                _updateLabel.setFrame({x:50, y:100, width:100, height:30})
                _updateLabel.setText("点击按钮更新JS代码--->")
                _updateLabel.setFont(require('UIFont').systemFontOfSize(15))
                _updateLabel.setTextColor(require('UIColor').redColor())
                _updateLabel.sizeToFit()
                self.setUpdateLabel(_updateLabel)
            }
            return _updateLabel
            },
})
```
代码中是有添加了`updateLabel`这样一个属性，并添加到`JSPatchController`的视图中，其中`JSPatchController`为window下navigationController的rootViewController。

当我尝试在检查更新的block没有更新的区块中执行

``` objectivec
[JPEngine startEngine];
 NSString *script = [NSString stringWithContentsOfFile:hotfixPath encoding:NSUTF8StringEncoding error:nil]; 
[JPEngine evaluateScript:script]
```
发现`JSPatchController`并没有添加上`updateLabel`，起初以为是没有回到主线程中更新视图，经过尝试并不是。当执行NSURLSessionDataTask的网络请求后，会新开启一个线程。而主线程继续执行编译`JSPatchController`，当新的线程中的网络请求完成了，然后加载了js脚本文件，此时已经不能再对`JSPatchController`动态添加`updateLabel`了。

* 第二点：`didFinishLaunchingWithOptions:` 的执行优先级是高于`applicationDidBecomeActive:`方法的，检查本地的js文件应发在前者中。

* 第三点： `applicationDidBecomeActive:` 方法中检测需要更新并下载了js脚本文件，但是~~只有下一次启动App的时候才能生效~~  在根控制器中的动态修改代码只能下一次启动App的时候才能生效。


#### 4. 其他时机手动更新JS文件的效果
其他时机这里指的是例如`UIControl`事件，点击按钮后更新js脚本，那么这个脚本文件中的代码何时生效呢？
在demo中的`JSPatchController`中  “更新JS” 这个按钮，点击执行的代码如下：

``` objectivec
- (IBAction)updateJS:(UIButton *)sender {
    sender.selected = !sender.isSelected;
    NSString *url = sender.isSelected ? jsfile1 : jsfile2;
    [sender setTitle:@"当前JS1" forState:UIControlStateSelected];
    [sender setTitle:@"当前JS2" forState:UIControlStateNormal];
    
    NSLog(@"下载的是%@",url);
    [HotFixManager downLoadHotFixJSfileWithURL:url completeHandle:^(BOOL status, id response, NSError *error) {
        if (!status){
            NSLog(@"下载出错\n%@", error.userInfo);
            return ;
        }
        NSLog(@"Hotfix文件更新成功");
        [JPEngine startEngine];
        NSString *script = [NSString stringWithContentsOfFile:response encoding:NSUTF8StringEncoding error:nil];
        [JPEngine evaluateScript:script];
    }];
}
```

点按按钮有两个js脚本文件可以进行切换，两个js文件中的区别部分为对下一个控制器中的lable文字和navigation title的控制 

jsfile1：

``` javascript
defineClass('SecondViewController', {
            viewDidLoad: function() {
            self.ORIGviewDidLoad()
            var label = self.myLabel()
            label.setText("这是在JS1中修改的文字")
            self.setTitle("JS1推出的页面")
            }
            })
```

jsfile2:

``` javascript
defineClass('SecondViewController', {
            viewDidLoad: function() {
            self.ORIGviewDidLoad()
            var label = self.myLabel()
            label.setText("这是在JS2中修改的文字")
            self.setTitle("JS2推出的页面")
            }
            })
```

测试结果是：更新的js脚本中对之后的页面的js修复代码是可以生效的，但是对之前的页面跟同级的页面是没有效果的。

其实这些，只要搞清楚原理就都能知道缘由和规律。
#### 5. JS调用OC方法中的几个坑

* 在js中 NSNumber不需要在处理，可直接当数值使用。
* NSRang 初始化：
`var range = {location: 0, length: senderName.length()}`;
* 无论变量还是方法，单下划线全部改为双下划线
* CGRect 取宽高， 直接`rect.width`, 不用`rect.size.width`。其他结构体类似
* js 中 YES 为 ture，NO 为false
* oc对象转js对象可操作`toJS()`，js对象转oc对象暂时没找到方法。
js内创建的字典为js对象，传入oc方法无效
* jspatch 不支持变参方法，如`stringWithFormat:`，可用js字符串方法或`NSMutableString`代替。

#### 6. JS脚本文件的版本控制管理

一般使用的js脚本文件都是从服务器下发过来，服务器的接口需要返回时候需要更新的参数，需要则下载于当前程序版本号对应版本的js脚本文件。对于多个版本的项目来说，js脚本文件最好也进行版本控制避免出错，可以在服务器单独建立js脚本文件的Git仓库来进行管理。

#### 7. 更多思考
* JS热修复的代码在下一次更新中应当使用原生的代码替换，不能超过一个版本。避免对JSPatch有过多的依赖
* 使用JS语法来调用OC的方法，没有代码自动补全显得非常吃力。作者bang开发了[Xcode插件：JSPatchX](https://github.com/bang590/JSPatchX)，然而Xcode8不支持插件了。。。
* 在一个很复杂的方法中,仅中间某一行代码需要修改,就要将整个方法用JS重写一遍,推介作者开发的Objective-C转JavaScript代码工具[JSPatch Convertor](https://github.com/bang590/JSPatchConvertor),但一些复杂的语法还是要人工修正

