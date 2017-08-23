---
title: iOS隐式动画
date: 2017-07-19 09:08:59
tags: [iOS, 学习笔记]
---

先来一段引用改变下对动画的世界观：
>Core Animation基于一个假设，说屏幕上的任何东西都可以（或者可能）做动画。 动画并不需要你在Core Animation中手动打开，相反需要明确地关闭，否则他会一直存在。

``` objctivec

 	CGFloat red = arc4random() / (CGFloat)INT_MAX;
    CGFloat green = arc4random() / (CGFloat)INT_MAX;
    CGFloat blue = arc4random() / (CGFloat)INT_MAX;

    self.colorLayer.backgroundColor = [UIColor colorWithRed:red green:green blue:blue alpha:1.0].CGColor;
    
```
上面这样的一段代码，没有设置任何动画，但是在运行的时候会发现颜色的变化并不是突变的，有一定的动画效果。这个动画怎么实现的，时间又是多少呢？ 这得先从事务说起。

<!--more-->

>事务是Core Animation用来包含一系列属性动画集合的机制，任何用指定事务去改变可以做动画的图层属性都不会立刻发生变化，而是当事务一旦提交的时候开始用一个动画过渡到新值。

>事务通过 CATransaction 类来做管理，使用+begin和+commit来入栈和出栈。

Core Animation在每个run loop周期中自动开始一次新的事务，即使不显式的调用`[CATransaction begin]` 开始一次事务，任何在一次run loop循环中属性的改变都会被集中起来，然后做一次__0.25__秒的动画。

### 关闭隐式动画

Core Animation通常对CALayer的所有属性（可动画的属性）做动画，但是UIView把它关联的图层的这个特性关闭了。

我们把改变属性时CALayer自动应用的动画称作行为，当CALayer的属性被修改时候，它会调用-actionForKey:方法，传递属性的名称。
具体看看是隐式动画的实现步骤：

这是`CALayer`头文件里对`actionForKey:`方法调用的描述

```
/* Returns the action object associated with the event named by the
 * string 'event'. The default implementation searches for an action
 * object in the following places:
 *
 * 1. if defined, call the delegate method -actionForLayer:forKey:
 * 2. look in the layer's `actions' dictionary
 * 3. look in any `actions' dictionaries in the `style' hierarchy
 * 4. call +defaultActionForKey: on the layer's class
 *
 * If any of these steps results in a non-nil action object, the
 * following steps are ignored. If the final result is an instance of
 * NSNull, it is converted to `nil'. */

- (nullable id<CAAction>)actionForKey:(NSString *)event;
```

1. 图层首先检测它是否有委托，并且是否实现CALayerDelegate协议指定的-actionForLayer:forKey方法。如果有，直接调用并返回结果。

2. 如果没有委托，或者委托没有实现-actionForLayer:forKey方法，图层接着检查包含属性名称对应行为映射的actions字典。

3. 如果actions字典没有包含对应的属性，那么图层接着在它的style字典接着搜索属性名。

4. 最后，如果在style里面也找不到对应的行为，那么图层将会直接调用定义了每个属性的标准行为的-defaultActionForKey:方法。

所以一轮完整的搜索结束之后，-actionForKey:要么返回空（这种情况下将不会有动画发生），要么是CAAction协议对应的对象，最后CALayer拿这个结果去对先前和当前的值做动画。

这就解释了UIKit是如何禁用隐式动画的：每个UIView对它关联的图层都扮演了一个委托，并且提供了-actionForLayer:forKey的实现方法。当不在一个动画块的实现中，UIView对所有图层行为返回nil，但是在动画block范围之内，它就返回了一个非空值。


ps : `[CATransaction begin]`之后添加下面的代码，同样也会阻止动画的发生：

		[CATransaction setDisableActions:YES];

### 关联图层设置动画

* UIView关联的图层禁用了隐式动画，对这种图层做动画的唯一办法就是使用UIView的动画函数（而不是依赖CATransaction），或者继承UIView，并覆盖-actionForLayer:forKey:方法，或者直接创建一个[显式动画]()_（等待下一篇笔记）_。


* 对于单独存在的图层，我们可以通过实现图层的-actionForLayer:forKey:委托方法，或者提供一个actions字典来控制隐式动画。 

### 自定义行为

行为通常是一个被Core Animation隐式调用的显式动画。


我们来对颜色渐变的代码（文章顶部）使用一个不同的行为，通过给colorLayer设置一个自定义的actions字典。我们也可以使用委托来实现，但是actions字典可以写更少的代码。

PS: CATransition响应CAAction协议，并且可以当做一个图层行为

``` objectivec
  //add a custom action
    CATransition *transition = [CATransition animation];
    transition.type = kCATransitionPush;
    transition.subtype = kCATransitionFromLeft;
    self.colorLayer.actions = @{@"backgroundColor": transition};
    
```

在初始化的时候给layer添加了这样的行为之后，就可以对隐式动画添加了一个 推进过度 的效果

### Reference

[iOS Core Animation](https://zsisme.gitbooks.io/ios-/content/chapter7/layer-actions.html) 十分感谢译者