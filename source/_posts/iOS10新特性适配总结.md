---
title: iOS10新特性适配总结
date: 2016-12-16 08:45:51
tags: iOS
---

### 1. Notification(通知)
自从Notification被引入之后，苹果就不断的更新优化，但这些更新优化只是小打小闹，直至现在iOS 10开始真正的进行大改重构，这让开发者也体会到UserNotifications的易用，功能也变得非常强大。

#### iOS 9 以前的通知

1. 在调用方法时，有些方法让人很难区分，容易写错方法，这让开发者有时候很苦恼。

2. 应用在运行时和非运行时捕获通知的路径还不一致。

3. 应用在前台时，是无法直接显示远程通知，还需要进一步处理。

4. 已经发出的通知是不能更新的，内容发出时是不能改变的，并且只有简单文本展示方式，扩展性根本不是很好。

####  iOS 10 开始的通知

1. 所有相关通知被统一到了UserNotifications.framework框架中。

2. 增加了撤销、更新、中途还可以修改通知的内容。

3. 通知不在是简单的文本了，可以加入视频、图片，自定义通知的展示等等。

4. iOS 10相对之前的通知来说更加好用易于管理，并且进行了大规模优化，对于开发者来说是一件好事。

5. iOS 10开始对于权限问题进行了优化，申请权限就比较简单了(本地与远程通知集成在一个方法中)。

如果使用了推送，修改如图：

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160919115440465-282261076.jpg)

### 2. ATS的问题
iOS 9中默认非HTTS的网络是被禁止的，当然我们也可以把NSAllowsArbitraryLoads设置为YES禁用ATS。不过iOS 10从2017年1月1日起苹果不允许我们通过这个方法跳过ATS，也就是说强制我们用HTTPS，如果不这样的话提交App可能会被拒绝。但是我们可以通过NSExceptionDomains来针对特定的域名开放HTTP可以容易通过审核。

NSExceptionDomains方式 设置域。可以简单理解成，把不支持https协议的接口设置成http的接口。

具体方法：

1. 在项目的info.plist中添加一个Key：App Transport Security Settings，类型为字典类型。

2. 然后给它添加一个Exception Domains，类型为字典类型；

3. 把需要的支持的域添加給Exception Domains。其中域作为Key，类型为字典类型。

4. 每个域下面需要设置3个属性：NSIncludesSubdomains、NSExceptionRequiresForwardSecrecy、NSExceptionAllowsInsecureHTTPLoads。

如图：
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918170238256-1807962290.png)
细节提示:在iOS9以后的系统中如果使用到网络图片，也要注意网络图片是否是HTTP的哦，如果是，也要把图片的域设置哦！

### 3. iOS 10 隐私权限设置
iOS 10 开始对隐私权限更加严格，如果你不设置就会直接崩溃，现在很多遇到崩溃问题了，一般解决办法都是在info.plist文件添加对应的Key-Value就可以了。
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918164446868-505879984.png)
以上Value值，圈出的红线部分的文字是展示给用户看的，必须添加。

### 4. Xcode 8 运行一堆没用的logs解决办法
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918114127228-2093692903.jpg)

上图我们看到，自己新建的一个工程啥也没干就打印一堆烂七八糟的东西，我觉得这个应该是Xcode 8的问题，

具体也没细研究，解决办法是设置OS_ACTIVITY_MODE : disable如下图:
第一步：
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160919101042684-1974264450.png)
第二步：
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160919101103902-448062410.png)
第三步：
添加参数：
Name ：`OS_ACTIVITY_MODE `
Value :  `disable`
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160919101202996-1273097948.png)

### 5. iOS 10 UIStatusBar方法过期:
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918114216825-2072979176.jpg)
在我们开发中有可能用到UIStatusBar一些属性，在iOS 10 中这些方法已经过期了，如果你的项目中有用的话就得需要适配。

上面的图片也能发现，如果在iOS 10中你需要使用preferredStatusBar比如这样：


	//iOS 10 - (UIStatusBarStyle)preferredStatusBarStyle {    return UIStatusBarStyleDefault;
	}	
 
	
###  6. iOS 10 UICollectionView 性能优化
随着开发者对UICollectionView的信赖，项目中用的地方也比较多，但是还是存在一些问题，比如有时会卡顿、加载慢等。所以iOS 10 对UICollectionView进一步的优化。

* UICollectionView cell pre-fetching预加载机制
* UICollectionView and UITableView prefetchDataSource 新增的API
* 针对self-sizing cells 的改进
* Interactive reordering

在iOS 10 之前,UICollectionView上面如果有大量cell,当用户活动很快的时候,整个UICollectionView的卡顿会很明显,为什么会造成这样的问题,这里涉及到了iOS 系统的重用机制,当cell准备加载进屏幕的时候,整个cell都已经加载完成,等待在屏幕外面了,也就是整整一行cell都已经加载完毕,这就是造成卡顿的主要原因,专业术语叫做:掉帧.
要想让用户感觉不到卡顿,我们的app必须帧率达到60帧/秒,也就是说每帧16毫秒要刷新一次.

#### iOS 10 之前UICollectionViewCell的生命周期是这样的:
1. 用户滑动屏幕,屏幕外有一个cell准备加载进来,把cell从reusr队列拿出来,然后调用prepareForReuse方法,在这个方法里面,可以重置cell的状态,加载新的数据;
2. 继续滑动,就会调用cellForItemAtIndexPath方法,在这个方法里面给cell赋值模型,然后返回给系统;
3. 当cell马上进去屏幕的时候,就会调用willDisplayCell方法,在这个方法里面我们还可以修改cell,为进入屏幕做最后的准备工作;
4. 执行完willDisplayCell方法后,cell就进去屏幕了.当cell完全离开屏幕以后,会调用didEndDisplayingCell方法.
#### iOS 10 UICollectionViewCell的生命周期是这样的:
1. 用户滑动屏幕,屏幕外有一个cell准备加载进来,把cell从reusr队列拿出来,然后调用prepareForReuse方法,在这里当cell还没有进去屏幕的时候,就已经提前调用这个方法了,对比之前的区别是之前是cell的上边缘马上进去屏幕的时候就会调用该方法,而iOS 10 提前到cell还在屏幕外面的时候就调用;
2. 在cellForItemAtIndexPath中创建cell，填充数据，刷新状态等操作,相比于之前也提前了;
3. 用户继续滑动的话,当cell马上就需要显示的时候我们再调用willDisplayCell方法,原则就是:何时需要显示,何时再去调用willDisplayCell方法;
4. 当cell完全离开屏幕以后,会调用didEndDisplayingCell方法,跟之前一样,cell会进入重用队列.
在iOS 10 之前,cell只能从重用队列里面取出,再走一遍生命周期,并调用cellForItemAtIndexPath创建或者生成一个cell.
在iOS 10 中,系统会cell保存一段时间,也就是说当用户把cell滑出屏幕以后,如果又滑动回来,cell不用再走一遍生命周期了,只需要调用willDisplayCell方法就可以重新出现在屏幕中了.
iOS 10 中,系统是一个一个加载cell的,二以前是一行一行加载的,这样就可以提升很多性能;
#### iOS 10 新增加的Pre-Fetching预加载
这个是为了降低UICollectionViewCell在加载的时候所花费的时间,在 iOS 10 中,除了数据源协议和代理协议外,新增加了一个UICollectionViewDataSourcePrefetching协议,这个协议里面定义了两个方法:



```objectivec
　　- (void)collectionView:(UICollectionView *)collectionView prefetchItemsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths NS_AVAILABLE_IOS(10_0;
　　
```



```objectivec
　　- (void)collectionView:(UICollectionView *)collectionView cancelPrefetchingForItemsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths  NS_AVAILABLE_IOS(10_0;
　　
```
　　　在ColletionView prefetchItemsAt indexPaths这个方法是异步预加载数据的,当中的indexPaths数组是有序的,就是item接收数据的顺序;
　　　CollectionView cancelPrefetcingForItemsAt indexPaths这个方法是可选的,可以用来处理在滑动中取消或者降低提前加载数据的优先级.
　　　注意:这个协议并不能代替之前读取数据的方法,仅仅是辅助加载数据.
　　　Pre-Fetching预加载对UITableViewCell同样适用.


### 7. iOS 10 UIColor 新增方法
以下是官方文档的说明：

> Most graphics frameworks throughout the system, including Core Graphics, Core Image, Metal, and AVFoundation, have substantially improved support for extended-range pixel formats and wide-gamut color spaces. By extending this behavior throughout the entire graphics stack, it is easier than ever to support devices with a wide color display. In addition, UIKit standardizes on working in a new extended sRGB color space, making it easy to mix sRGB colors with colors in other, wider color gamuts without a significant performance penalty.

> Here are some best practices to adopt as you start working with Wide Color.

> In iOS 10, the UIColor class uses the extended sRGB color space and its initializers no longer clamp raw component values to between 0.0 and 1.0. If your app relies on UIKit to clamp component values (whether you’re creating a color or asking a color for its component values), you need to change your app’s behavior when you link against iOS 10.

> When performing custom drawing in a UIView on an iPad Pro (9.7 inch), the underlying drawing environment is configured with an extended sRGB color space.

> If your app renders custom image objects, use the new UIGraphicsImageRenderer class to control whether the destination bitmap is created using an extended-range or standard-range format.

> If you are performing your own image processing on wide-gamut devices using a lower level API, such as Core Graphics or Metal, you should use an extended range color space and a pixel format that supports 16-bit floating-point component values. When clamping of color values is necessary, you should do so explicitly.

> Core Graphics, Core Image, and Metal Performance Shaders provide new options for easily converting colors and images between color spaces.

因为之前我们都是用RGB来设置颜色，反正用起来也不是特别多样化，这次新增的方法应该就是一个弥补吧。所以在iOS 10 苹果官方建议我们使用sRGB，因为它性能更好，色彩更丰富。如果你自己为UIColor写了一套分类的话也可尝试替换为sRGB，UIColor类中新增了两个Api如下:

``` objectivec
+ (UIColor *)colorWithDisplayP3Red:(CGFloat)displayP3Red green:(CGFloat)green blue:(CGFloat)blue alpha:(CGFloat)alpha NS_AVAILABLE_IOS(10_0);

- (UIColor *)initWithDisplayP3Red:(CGFloat)displayP3Red green:(CGFloat)green blue:(CGFloat)blue alpha:(CGFloat)alpha NS_AVAILABLE_IOS(10_0);

```

### 8. iOS 10 UITextContentType

	// The textContentType property is to provide the keyboard with extra information about the semantic intent of the text document.@property(nonatomic,copy) UITextContentType textContentType NS_AVAILABLE_IOS(10_0); // default is nil
在iOS 10 UITextField添加了textContentType枚举，指示文本输入区域所期望的语义意义。

使用此属性可以给键盘和系统信息，关于用户输入的内容的预期的语义意义。例如，您可以指定一个文本字段，用户填写收到一封电子邮件确认uitextcontenttypeemailaddress。当您提供有关您期望用户在文本输入区域中输入的内容的信息时，系统可以在某些情况下自动选择适当的键盘，并提高键盘修正和主动与其他文本输入机会的整合。

 
### 9. iOS 10 字体随着手机系统字体而改变
当我们手机系统字体改变了之后，那我们App的label也会跟着一起变化，这需要我们写很多代码来进一步处理才能实现，但是iOS 10 提供了这样的属性adjustsFontForContentSizeCategory来设置。因为没有真机，具体实际操作还没去实现，如果理解错误帮忙指正。

	  UILabel *myLabel = [UILabel new];   /*
   UIFont 的preferredFontForTextStyle: 意思是指定一个样式，并让字体大小符合用户设定的字体大小。

``` objectivec
  myLabel.font =[UIFont preferredFontForTextStyle: UIFontTextStyleHeadline];
  /*
	 Indicates whether the corresponding element should automatically update its font when the device’s UIContentSizeCategory is changed.
	 For this property to take effect, the element’s font must be a font vended using +preferredFontForTextStyle: or +preferredFontForTextStyle:compatibleWithTraitCollection: with a valid UIFontTextStyle.
	 */
```
	 
//是否更新字体的变化

	    myLabel.adjustsFontForContentSizeCategory = YES;
 

### 10. iOS 10 UIScrollView新增refreshControl

iOS 10 以后只要是继承UIScrollView那么就支持刷新功能：

	@property (nonatomic, strong, nullable) UIRefreshControl *refreshControl NS_AVAILABLE_IOS(10_0) __TVOS_PROHIBITED;


### 11. iOS 10 判断系统版本正确姿势
判断系统版本是我们经常用到的，尤其是现在大家都有可能需要适配iOS 10，那么问题就出现了，如下图：
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918114620576-1303627103.jpg)
我们得到了答案是：

//值为 1 `[[[[UIDevice currentDevice] systemVersion] substringToIndex:1] integerValue]`

//值为10.000000 `[[UIDevice currentDevice] systemVersion].floatValue`,

//值为10.0` [[UIDevice currentDevice] systemVersion]`

所以说判断系统方法最好还是用后面的两种方法，哦~我忘记说了`[[UIDevice currentDevice] systemVersion].floatValue`这个方法也是不靠谱的，好像在8.3版本输出的值是8.2，记不清楚了反正是不靠谱的，所以建议大家用`[[UIDevice currentDevice] systemVersion]`这个方法！

Swift判断如下：
``` swift
  if #available(iOS 10.0, *) {
            // iOS 10.0
            print("iOS 10.0");
        } else { }
```

### 12. Xcode 8 插件不能用的问题
大家都升级了Xcode 8，但是对于插件依赖的开发者们，一边哭着一边去网上寻找解决办法。那么下面是解决办法：
让你的 Xcode8 继续使用插件(http://vongloo.me/2016/09/10/Make-Your-Xcode8-Great-Again/?utm_source=tuicool&utm_medium=referral )

但是看到文章最后的解释，我们知道如果用插件的话，可能安全上会有问题、并且提交审核会被拒绝，所以建议大家还是不要用了，解决办法总是有的，比如在Xcode中添加注释的代码块也是很方便的。

 

### 13. iOS 10开始项目中有的文字显示不全问题
我用Xcode 8 和Xcode 7.3分别测试了下，如下图：
![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918114911979-1889192985.jpg)

Xcode 8

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918114920417-1828331146.jpg)

Xcode7 

创建一个Label然后让它自适应大小，字体大小都是17最后输出的宽度是不一样的，我们再看一下，
下面的数据就知道为什么升级iOS 10 之后App中有的文字显示不全了：

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918115047642-1072358093.png)

英文字母会不会也有这种问题，我又通过测试，后来发现英文字母没有问题，只有汉字有问题。
目前只有一个一个修改控件解决这个问题，暂时没有其他好办法来解决。

### 14.  Xcode 8使用Xib awakeFromNib的警告问题
在Xcode 8之前我们使用Xib初始化`- (void)awakeFromNib {}`都是这么写也没什么问题，但是在Xcode 8会有如下警告：

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160918164934170-318856946.png)

官方解释：
> You must call the super implementation of awakeFromNib to give parent classes the opportunity to perform any additional initialization they require.
> Although the default implementation of this method does nothing, many UIKit classes provide non-empty implementations. 
> You may call the super implementation at any point during your own awakeFromNib method.


你必须调用父类实现`awakeFromNib`来给父类来执行它们需要的任何额外的初始化的机会。
虽然这种方法的默认实现不做任何事情，许多UIKit类提供非空的实现。
你可以调用自己的`awakeFromNib`方法中的任何时候超级实现。

### 15. 推送的时候，开启`Remote notifications`

	You've implemented -[<UIApplicationDelegate> application:didReceiveRemoteNotification:fetchCompletionHandler:],
	but you still need to add "remote-notification" to the list of your supported UIBackgroundModes in your Info.plist.*
解决方案：需要在Xcode 中修改应用的 Capabilities 开启Remote notifications，请参考下图：

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160919140245793-794728449.png)


### 16.  `One of the two will be used. Which one is undefined.`
	objc[5114]: Class PLBuildVersion is implemented in both /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/AssetsLibraryServices.framework/AssetsLibraryServices (0x1109a5910) and /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/PrivateFrameworks/PhotoLibraryServices.framework/PhotoLibraryServices (0x110738210). One of the two will be used. Which one is undefined.

　　在模拟器中、发现“One of the two will be used. Which one is undefined.”日志

　　查找资料发现原因：objc runtime 对所用app使用同一个命名空间(flat namespace)，运行机制如下：
1. 首先二进制映像被加载，检查程序依赖关系
2. 每一个二进制映像被加载的同时，程序的objc classes在objc runtime命名空间中注册
3. 如果具有相同名称的类被再次加载，objc runtime的行为是不可预知的。一种可能的情况是任意一个程序的该类会被加载(这应该也是默认动作)

### 17. `Invalid Bundle - The asset catalog at 'Payload/XXXXX/Assets.car' can't contain 16-bit or P3 assets if the app supports iOS 9.3 or earlier`

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160920180122621-1767554257.png)

在 Xcode 8 中，当你资源文件中[含有16位图]或者[图片显示模式γ值为'P3']且iOS targets设定为iOS 9.3以下就会出现这个问题. 如果你的app需要支持广色域显示的话，那你必须得把target设置成iOS 9.3+，相反，如果你的app不需要支持广色域且你想兼容 iOS 9.3 之前的项目，你就得把所有的16位的或者显示模式为'P3'图片全都替换成8位模式的SRGB颜色的图片。

 

你可以通过运行“assetutil”在iTunes Connect的错误信息中找到16-bit 或 P3 资源文件。离线的解决方案如下：

1. 导出项目的 ipa 文件

2. 定位到该ipa文件修改后缀名.ipa 为 .zip.

3.  解压该 .zip 文件. 解压后的目录里面会有一个包含着你的 app bundle 文件的 Payload 文件夹.

4. 打开终端病切换到你的app的Payload文件夹下的 .app bundle 文件夹内，形式如下：

`cd path/to/Payload/your.app`

5. 用 find 命令定位到 Assets.car 文件 .app bundle , 形式如下:

`find . -name 'Assets.car'`

6. 使用 assetutil 命令找到任何包含着 16-bit or P3 的资源文件, 对每个 Assets.car 之行以下命令 :

`sudo xcrun --sdk iphoneos assetutil --info /path/to/a/Assets.car > /tmp/Assets.json`

7.  打开上一步生成的 /tmp/Assets.json 文件并查找包含有 “DisplayGamut": “P3” 或者相关的内容.  这段json的"Name"字段对应的值就是16位或显示的γ值为P3的资源文件名.

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160920180323184-1049564596.png)

8.  找到这个资源文件修改为 8位的sRGB形式,重新编译上传你的app即可. 


### 18. This version does not support documents saved in the Xcode 8 format. Open this document with Xcode 8 or later
　 编辑项目时默认使用Xcode8打开，导致我用Xcode7打开Xib是报错：

	 This version does not support documents saved in the Xcode 8 format. Open this document with Xcode 8.0 or later

![](http://images2015.cnblogs.com/blog/575661/201609/575661-20160921142554277-537167035.png)

   导致用Xcode8打开的Xib全部打不开，只能用编辑器将Xib里面的下面一句话删除掉才能打开：

	<capability name="documents saved in the Xcode 8 format" minToolsVersion="8.0"/>

