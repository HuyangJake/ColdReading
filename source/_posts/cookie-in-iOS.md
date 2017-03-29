---
title: cookie in iOS
date: 2017-01-12 16:37:13
tags: iOS
---

### 背景
项目中用到了本地登录存储cookie，再进行webView请求时间带上cookie的功能。不是很清晰逻辑，于是学习了解了下iOS中cookie的使用，做个小结。跳过概念直接看[使用🌰](#example)

<!-- more -->

### Cookie介绍
Cookie分类两类：

* 会话cookie
* 持久cookie

会话 cookie 和持久 cookie 之间唯一的区别就是它们的过期时间。

__cookie如何工作__

cookie 中包含了一个由 名字 = 值 （name=value） 这样的信息构成的任意列表，并通过 Set-Cookie 或 Set-Cookie2 HTTP 响应（扩 展）首部将其贴到用户身上去。
![cookie工作](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/cookie%E5%B7%A5%E4%BD%9C.png-blogwebp)


__cookie组成__
现在使用的 cookie 规范有两个不同的版本：cookies 版本 0（有时被称为 Netscape cookies） 和 cookies 版 本 1（RFC 2965） 。 cookies 版 本 1 是 对 cookies 版 本 0 的 扩 展，应用不如后者广泛。

* __cookie版本0__
	* Set-Cookie 首部
Set-Cookie 首部有一个强制性的cookie名和cookie值。后面跟着可选的cookie 属性，中间由分号分隔。
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/setcookie%E9%A6%96%E9%83%A81.png-blogwebp)
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/setcookie%E9%A6%96%E9%83%A82.png-blogwebp)

	* cookie 0首部:

```
Cookie: session-id=002-1145265-8016838; session-id-time=1007884800
```

* __cookie版本1__ 
	* Set-Cookie2首部
这个版本 1 标准引入了Set-Cookie2首部和Cookie2首部，但它也能于版本0进行交互操作。
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/setcookie2%E9%A6%96%E9%83%A8.png-blogwebp)

	* cookie 1首部:

```
Cookie: $Version="1";
ID="29046"; $Domain=".joes-hardware.com";
color="blue";
Coupon="hammer027"; $Path="/tools";
Coupon="handvac103"; $Path="/tools/cordless"
```

### NSHTTPCookieStorage

> NSHTTPCookieStorage implements a singleton object (shared instance) that manages storage of cookies. Each cookie is represented by an instance of the NSHTTPCookie class. As a rule, cookies are shared among all applications and are kept in sync across process boundaries. Session cookies (where the cookie object’s sessionOnly method returns YES) are local to a single process and are not shared.

`NSHTTPCookieStorage`的实现是一个单例对象，管理着`NSHTTPCookie`对象，会话cookie的 `sessionOnly`方法返回`YES`就不可在进程间共享。

然而苹果在这个类的API Reference中写了一个`iOS Note`：
>Cookies are not shared among applications in iOS.

* __iOS__中cookie不能跨应用共享

* __MacOS__中`sessionOnly`不为`YES`是可以被共享的


`NSHTTPCookieStorage`可以管理cookie的接受策略，在一个app中改变cookie的接受策略将会影响其他正在运行的app的cookie接受策略。

当其他的app修改了cookie存储 或者 cookie接受策略，NSHTTPCookieStorage将会给app发送`NSHTTPCookieManagerCookiesChangedNotification`或者`NSHTTPCookieStorageAcceptPolicyChangedNotification`通知


__Cookie接受策略：__
`NSHTTPCookieAcceptPolicy`

``` objectivec
typedef NS_ENUM(NSUInteger, NSHTTPCookieAcceptPolicy) {
    NSHTTPCookieAcceptPolicyAlways,//默认策略，接受所有的cookies
    NSHTTPCookieAcceptPolicyNever,//拒绝所有的cookies
    NSHTTPCookieAcceptPolicyOnlyFromMainDocumentDomain//只从主文档域名接受cookies
};
```


### NSHTTPCookie

>An NSHTTPCookie object represents an HTTP cookie. It is an immutable object, initialized from a dictionary containing the cookie attributes.

`NSHTTPCookie`对象中包含了HTTP 的cookie对象，从包含cookie字段的字典初始化创建。

>The NSHTTPCookie class encapsulates a cookie, providing accessors for many of the common cookie attributes. This class also provides methods to convert HTTP cookie headers to NSHTTPCookie instances and convert an NSHTTPCookie instance to headers suitable for use with an NSURLRequest object. The URL loading system automatically sends any stored cookies appropriate for an NSURLRequest object unless the request specifies not to send cookies. Likewise, cookies returned in an NSURLResponse object are accepted in accordance with the current cookie acceptance policy.

`NSHTTPCookie`类封装了一个HTTP Cookie，并提供常见的Cookie属性访问接口。`NSHTTPCookie`可以转换HTTP Cookie成NSHTTPCookies对象，并可以将NSHTTPCookie对象转化为NSURLRequest对象的请求头部分。URL loading system 会自动地给NSURLRequest对象发送存储的cookie，除非NSURLRequest对象指定不需要传cookie，同样cookie在NSURLRequest中返回也按照当前的cookie接受策略接收。



### <span id = "example">Cookie使用💪个🌰</span>

Cookie生成的有两个途径，一个是访问一个网页，这个网页返回的HTTP Header中有Set-Cookie指令进行Cookie的设置，这里Cookie的本地处理其实是由WebKit进行的；还有一种途径就是客户端通过代码手动设置Cookie。

值得注意[iOS Cookie使用](http://geeklu.com/2013/04/ios-cookie/)提到：
NSHTTPCookieStorage存在一个问题，setCookie或者deleteCookie后并不会立即进行持久化，而是有几秒的延迟。为了可靠性，我们会将cookie信息保存一份到User Defaults，需要用的时候load进来。

* 客户端手动设置Cookie

``` objectivec
NSMutableDictionary *cookieProperties = [NSMutableDictionary dictionary];
[cookieProperties setObject:@"name" forKey:NSHTTPCookieName];
[cookieProperties setObject:@"value" forKey:NSHTTPCookieValue];
[cookieProperties setObject:@"www.taobao.com" forKey:NSHTTPCookieDomain];
[cookieProperties setObject:@"/" forKey:NSHTTPCookiePath];
[cookieProperties setObject:@"0" forKey:NSHTTPCookieVersion];
[cookieProperties setObject:@"30000" forKey:NSHTTPCookieMaximumAge];
NSHTTPCookie *cookie = [NSHTTPCookie cookieWithProperties:cookieProperties];
[[NSHTTPCookieStorage sharedHTTPCookieStorage] setCookie:cookie];
//删除cookie的方法为deleteCookie:

```

在通过setCookie:进行设置cookie的时候，会覆盖name,domain,path都相同的cookie的。 
至于cookie会不会持久化到cookie文件中主要看这个cookie的生命周期，和Max-Age或者Expires有关。

* 通过HTTP Header的Set-Cookie后者Set-Cookie2设置Cookie

``` objectivec
 //请求一个网址，即可分配到cookie
    AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
    manager.responseSerializer = [AFJSONResponseSerializer new];
    [manager GET:urlString parameters:nil success:^(AFHTTPRequestOperation *operation, id responseObject) {
 
        //获取cookie
        NSArray *cookies = [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookies];
        
        /*
         * 把cookie进行归档并转换为NSData类型
         * 注意：cookie不能直接转换为NSData类型，否则会引起崩溃。
         * 所以先进行归档处理，再转换为Data
         */
        NSData *cookiesData = [NSKeyedArchiver archivedDataWithRootObject: [[NSHTTPCookieStorage sharedHTTPCookieStorage] cookies]];
 
        //存储归档后的cookie
        NSUserDefaults *userDefaults = [NSUserDefaults standardUserDefaults];
        [userDefaults setObject: cookiesData forKey: @"cookie"];
        [self setCookie];
        
    } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
 
    }];
    
    - (void)setCookie
{
    //取出保存的cookie
    NSUserDefaults *userDefaults = [NSUserDefaults standardUserDefaults];
 
    //对取出的cookie进行反归档处理
    NSArray *cookies = [NSKeyedUnarchiver unarchiveObjectWithData:[userDefaults objectForKey:@"cookie"]];
 
    if (cookies) {
        //设置cookie
        NSHTTPCookieStorage *cookieStorage = [NSHTTPCookieStorage sharedHTTPCookieStorage];
        for (id cookie in cookies) {
            [cookieStorage setCookie:(NSHTTPCookie *)cookie];
        }
    }else{
        NSLog(@"无cookie");
   }  
}
```




### 参考

[NSHTTPCookieStorage](https://developer.apple.com/reference/foundation/nshttpcookiestorage)

[Cookies and Custom Protocols](https://developer.apple.com/library/prerelease/content/documentation/Cocoa/Conceptual/URLLoadingSystem/CookiesandCustomProtocols/CookiesandCustomProtocols.html)

[http://geeklu.com/2013/04/ios-cookie/](http://geeklu.com/2013/04/ios-cookie/)

[http://blog.it985.com/11248.html](http://blog.it985.com/11248.html)

_《HTTP 权威指南》第11章 客户端识别与cookie机制_

### 相关导引

解决UIWebView和WKWebView之间的cookie共享问题：

[iOS开发-打通UIWebView和WKWebView的Cookie](https://fengqiangboy.com/14611518603473.html)