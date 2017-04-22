---
title: H5跳转Native路由机制的探索
date: 2017-03-24 09:20:47
tags: iOS
---

公司的项目核心业务都在H5，为了提高APP的使用体验，这段时间在重构部分功能业务到Native。重构过程持续了4期（_一期1~2周_），天天工作量杠杠的（_因为追求速度，线上质量打了折扣，现在放慢脚步正在稳固，我才得以写这篇文章，接下来还要继续下一期的重构,又期待又紧张_）。在重构过程中，收获到了挺多Native与H5之间交互和一些动态配置的体会，就在这记录下这些风雨历程吧。

<!--more-->
ps: 文中不涉及具体代码，主要记录实现的机制和逻辑，之后将代码方案抽离项目后会添加demo地址

### 功能需求
扯一扯：
公司现在版本的APP的主业务在H5端，那么我们Native重构的页面就避免不了跟H5之间的交互。这是最基本的需求，完成这个需求的方案已经比较稳定，我们使用自己定义的协议对需要交互的功能进行拦截实现H5到Native的交互。Native到H5的话就直接在控制器内加载指定地址的webView。

很显然，在Native到H5这一步有很大的优化空间。老大提出了这样的要求：

*  所有跳转的H5地址都需要能够在服务端动态配置，下文称：[__DymaticH5方案__](#DymaticH5)
*  所有的Native页面能够被H5调起，不管在应用内还APP外，下文称：[__NativeScheme方案__](#NativeScheme)

另外一位H5同事附加提出了一套线上容错方案(_拯救了两次线上bug_)

* 所有(_有需求_)的Native页面都能够通过服务端配置随时替换成H5， 下文称：[__SwitchPage方案__](#SwitchPage)

### 方案探索

##### <span id = "DymaticH5">DymaticH5方案</span>
首先介绍下环境：

我们的在线环境配置文件都放在自己实现的静态接口平台下面称`tms`，由我们前端部门的同学自己管理，可以方便及时地配合客户端和H5的迭代和更新。

*  首先我们将所有需要动态化配置(_由Native跳到H5_)的H5地址页面地址整理放到`tms`的配置文件中，例如:

``` 
{
	home : https://www.baidu.com,
	mine : https://www.mine.com,
	news : https://www.news.com
}

```

* 工程项目中添加与`tms`配置文件格式相同的`plist`文件,用于存储当`tms`上没有配置动态H5地址时的默认H5地址。

* 整个项目中的H5地址由`URLCenter`工具类集中管理，获得H5地址后存储在`DynamicH5Model`中。
	*  `DynamicH5Model`职责：
		* 获取plist中的H5地址给model赋值
		* 读取`tms`返回的H5地址给model重新赋值 
		
	* `URLCenter`职责：
	 	* 读取DynamicH5Model中的H5地址
	 	* 获取调用者传入H5地址所需要带的参数
	 	* 拼接参数返回调用者需要的完整的H5地址
	 	
我们来看一张简单的流程图：	

![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/DynamicH5%E6%96%B9%E6%A1%88.png-blogwebp)

APP启动程序先读取plist中的默认H5地址赋值到model，在请求tms配置文件之后，再获取tms中配置的H5地址赋值到model，这样可以保证每个H5地址都可以有值，并且都是我们所需要的最新的地址。如此，即使在线上，不用发版本我们的APP也可以自如地控制和更新每个功能模块跳转的H5页面内容。

当然，每个H5地址需要什么参数还是得调用者自己知道，然后将参数赋值到URLCenter中。

##### <span id = "NativeScheme">NativeScheme方案 </span>
取名叫NativeScheme，当然这个方案跟sheme有关，但也不仅仅是使用sheme。

先说明这个方案最终达到的效果吧：
	
APP中的部分功能模块点击跳转指向本地的哪个页面由服务端控制，不再需要本地固定。
可以随时切换功能模块跳转到对应的H5页面还是对应的Native页面，当然前提是这两者页面都有实现。

例如：
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/hospitalFunctions.png-blogwebp)

在这样的功能块中，每个功能模块点击后通过服务器返回的NativeScheme判断跳转到本地的对应页面。

__实现原理：__	

* 服务器给每个功能块添加一个nativeScheme的字段, 	如 `myApp://jakeApp/home` 

	其中	`myApp`为APP的info.plist配置文件中的URL Scheme

	 若部分页面初始化时有些必要的参数需要指定，NativeScheme可以附加必要的参数，如`myApp://jakeApp/home?homeType=1`
	 
* 工程项目中创建一个`nativeScheme.plist`，其中存储的是每个nativeScheme对应的本地控制器的名字和控制器初始化的方式（_`storyboard`或`xib`或`frame`_）

	如：
	
	```
	{
		myApp://jakeApp/home : {
		 class : HomeViewController,
		 type  : storyboard
		},
		
		myApp://jakeApp/mine : {
			class : MineViewController,
			type  : xib
		}
	}
	
	```

* 整个项目的控制器跳转由`URLRouter`控制，可传入H5地址或者NativeScheme地址获得将要跳转到的控制器
	* `URLRouter`职责:
		* 传入navtiveScheme格式的地址，则从本地`navtiveScheme.plist`中获取对应控制器并初始化传入从地址中获得的参数
		* 传入H5地址则获得完整的H5地址(若有使用第一节中的`DynamicH5方案`可以将其传入`URLCenter`获得地址)，创建webView跳转

* APP可通过 URL Scheme唤起，在AppDelegate对应的方法中使用`URLRouter`跳转到对应的Native页面

方案扩展使用：

我们还可以在nativeScheme判读环节添加一个动态的节点(_根据需求_)，以实现点击功能块后是跳转到H5页面还是跳转本地的页面(_此逻辑不同于本章第三节 “SwitchPage方案”_)。可以给每个功能块分别添加 本地scheme地址 和 H5地址 两个字段，用于分别是跳向哪种界面。判断规则可以根据自己的需求来定。

附上NativeScheme的简单流程图：
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/NativeScheme.png-blogwebp)


_此方案部分参考：[https://github.com/DarielChen/DCURLRouter](https://github.com/DarielChen/DCURLRouter) 感谢作者_

##### <span id = "SwitchPage">SwitchPage方案</span>
这套方案的存在，在某种意义上是一套线上的紧急备用方案----当重构的Native页面出现问题，可以通过此方案将该Native页面替换为H5页面。此方案，可以说是对我们项目现状
比较适用，在其他项目中或许存在的意义没有那么大。但是这套原理可以拿来做参考，毕竟SwitchPage可以将任意一个Native页面替换为任意一个H5的页面。

__实现原理__：

实现逻辑也比较简单，

 * `tms`上配置需要替换的控制器名字和需要替换成的H5地址的绝对路径，如： 

```
{
	HomeViewController : https://www.baidu.com
}
```

* 重写本地的Nativgation的push方法， 在此方法调用super之前，判断将要跳转的控制器名称在tms请求返回的数据中是否存在，如果存在则跳转webView控制器，不存在则按照原来方法跳转。（可结合NativeScheme方案使用，在URLRouter中实现此逻辑）

附上简单流程图：
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/SwitchPage.png-blogwebp)

### 总结
不足：NativeScheme方案中的参数传递比较地生硬，若是需要跳转的Native控制器需要制定的参数需要额外处理，则无法通过shceme后带参数的方式在`URLRouter`中直接给要返回的控制器添加参数


上面三套方案可以在项目中一起使用，相互之间不会影响，也可以单独使用其中的某一套方案。

demo地址：[即将出现]()