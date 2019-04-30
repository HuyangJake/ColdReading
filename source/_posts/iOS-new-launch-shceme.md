---
title: iOS开屏广告方案 （idea vs reality）
date: 2019-03-15 19:04:51
tags: [iOS]
---

__开屏广告__  emmm... 是个老生常谈的功能需求了。脑子里印象就是个很简单的需求，YY了一下实现的方式，我想到的方案大概是：

>在 `didFinishLaunchingWithOptions` 方法中切换`window`的`rootViewController`为启动广告页面，倒计时完成之后再切为正常的 `rootViewController`，其中再注意下处理图片缓存的存储和读取的操作。
>
>至于展示效果，加点动效的事情而已啦。修改rootViewController效果不好的话，想办法在window上加一个view展示咯，再不行使用一个新的window咯

<!-- more -->

终于在项目的这一次需求中要添加启动广告，我可以实现脑子里YY的功能了。
这里先列一下需求方给的详细需求：

> 1. 启动广告只需要在App启动的时候展示，倒计时3秒
> 2. App支持读取缓存展示启动广告、异步更新广告
> 3. 控制App是否强制展示此次返回的启动广告
> 4. 点击广告跳转到活动落地页，落地页返回到App首页

作为处女座的我，针对细节私自定了些额外的需求：
> a. 开屏广告上不展示状态栏
> b. 广告展示完成之后跳转到首页不那么突兀(首页需要已经加载完成)
> 

### 方案实现

假设如下是服务端返回的开屏广告控制接口。

``` json
{
	"code": 200,
	"msg": "SUCCESS",
	"data": {
		"hasAd": true,
		"id": 231,
		"imgUrl": "http://jakey.test.png",
		"jumpType": "url",
		"jumpValue": "https://www.baidu.com/",
		"showMoment": 0
	}
}
```

我创建了一个`LaunchAdManager`很快实现了切换`rootViewController`的方案，通过`SDWebImage`下载缓存图片，使用服务端的唯一字段`id`当做key缓存到内存和磁盘，不详细描述方案，具体流程见我的时序图，如下图：


![pi](http://note.huyangjie.cn/media/15526478756852/pic1.png)


最后的效果非常地生硬，两个rootViewController之间的切换就像是在打架...

---

### 寻找新方案

既然这样只能往 __在首页所在的window上添加新的view__ 的方案上进行靠了。 方案的对比优劣可以在 参考列表中查看[[2]](#ref)

我想到首页会有一个加载数据的过程，需要控制view在window上展示会不会跟其他的高优先级的弹窗冲突。以及这样会不会对业务会有所影响。

还是像键盘一样通过一个新的window对象展示出我们的启动广告就不会对原有业务有什么影响。

因为启动页是否展示需要请求接口，展示读取在线图片或是缓存图片也会需要一点时间。另外还需要有一点考虑的是，原本业务的首页需要在什么时机进行展示。如果太早，App启动会先进入业务首页，再出现开屏广告。如果太晚，开屏广告完成之后还没有加载完首页的数据，又达不到我定的需求 __b__ 。所以我设计了如下接口：

``` objectivec
/**
 检查是否需要展示广告

 @param completeHandler 判断接口请求完成
 @param finishedAction 倒计时完成
 */
- (void)checkIfNeedLaunchAdLoadComplete:(void(^)())completeHandler
                          countFinished:(void(^)(BOOL isSkiped))finishedAction;
```

在开屏广告接口请求完成之后就进行加载首页，此时新的window也已经被加载出来了，就不会出现前后不衔接的情况。

简单列一下关键代码

``` objectivec
UIWindow *adWindow = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
THKLaunchClearViewController *vc = [[THKLaunchClearViewController alloc] init];
adWindow.rootViewController = vc;
adWindow.windowLevel = UIWindowLevelStatusBar + 1;
adWindow.hidden = NO;
adWindow.alpha = 1;
[adWindow addSubview:rootVC.view];
```

关于 UIWindow 有只要创建了，它的hidden属性默认就是YES。在`makeKeyAndVisible`方法的注释中有这么写：

>Shows the window and makes it the key window.

>This is a convenience method to show the current window and position it in front of all other windows at the same level or lower. If you only want to show the window, change its hidden property to NO.
>

我们只是展示这个新的window，不是将其替换为keywindow, 那么只需要将其hidden的属性置为NO。

在展示完成之后，记得将新创建的window对象上的所有视图和其本身进行销毁。

---

想要直接使用现成的开屏广告组件？推荐使用文章底部参考链接中的 [[1]](#ref) 三方库，看了其实现的源码，虽然写得比较重稍显啰嗦，但功能全、效果不错，其star数经得起实战考验了。

在源码中看到了一种无痛接入的实现方式，先解释下无痛是什么概念：使用三方库不需要额外写任何代码，只要将库文件拖入工程就完成功能的实现。

具体实现的代码如下，通过重写类的load方法，因为类的load方法必定会执行，所以可以确保里面的代码执行。

``` objectivec
+(void)load{
    [self shareManager];
}

+(XHLaunchAdManager *)shareManager{
    static XHLaunchAdManager *instance = nil;
    static dispatch_once_t oneToken;
    dispatch_once(&oneToken,^{
        instance = [[XHLaunchAdManager alloc] init];
    });
    return instance;
}

- (instancetype)init{
    self = [super init];
    if (self) {
        //在UIApplicationDidFinishLaunching时初始化开屏广告,做到对业务层无干扰,当然你也可以直接在AppDelegate didFinishLaunchingWithOptions方法中初始化
        [[NSNotificationCenter defaultCenter] addObserverForName:UIApplicationDidFinishLaunchingNotification object:nil queue:nil usingBlock:^(NSNotification * _Nonnull note) {
            //初始化开屏广告
            [self setupXHLaunchAd];
        }];
    }
    return self;
}

```


### <span id="ref">参考</span>
[1].[XHLaunchAd GitHub](https://github.com/CoderZhuXH/XHLaunchAd)
[2]. [iOS开屏广告&弹窗浮层解决方案](https://my.oschina.net/zhxx/blog/910836)