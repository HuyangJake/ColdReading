---
title: iOS动画缓冲
date: 2017-10-09 23:28:26
tags: [iOS, ]
categories: [猿猿养成记, 学习笔记]
---

#### Core Animation内嵌了一系列标准函数来模拟物体在运动中的加速和减速。

### CAMediaTimingFunction

设置缓冲方程式首先需要设置`CAAnimation`的`timingFunction`属性，是`CAMediaTimingFunction`类的一个对象。

CAMediaTimingFunction的工厂方法：`+timingFunctionWithName:` 的参数常量如下：

```
kCAMediaTimingFunctionLinear 

kCAMediaTimingFunctionEaseIn 

kCAMediaTimingFunctionEaseOut 

kCAMediaTimingFunctionEaseInEaseOut

kCAMediaTimingFunctionDefault

```

* `kCAMediaTimingFunctionLinear`（线性计时函数） 是 `CAAnimation` 的 `timingFunction` 属性的默认函数。

* `kCAMediaTimingFunctionEaseIn` 常量创建了一个慢慢加速然后突然停止的方法。

* `kCAMediaTimingFunctionEaseOut` 则恰恰相反，它以一个全速开始，然后慢慢减速停止。

* `kCAMediaTimingFunctionEaseInEaseOut` 创建了一个慢慢加速然后再慢慢减速的过程。

* 最后还有一个 `kCAMediaTimingFunctionDefault` ，它和 `kCAMediaTimingFunctionEaseInEaseOut` 很类似，但是加速和减速的过程都稍微有些慢。

**当创建显式的CAAnimation，kCAMediaTimingFunctionDefault并不是默认选项**

**在UIView的动画中，kCAMediaTimingFunctionEaseInEaseOut 相对应的常量是默认效果**

### 缓冲和关键帧动画
`CAKeyframeAnimation`有一个NSArray类型的`timingFunctions`属性，我们可以用它来对每次动画的步骤指定不同的计时函数。但是指定函数的个数一定要等于`keyframes`数组的元素个数减一，因为它是描述每一帧之间动画速度的函数。

``` objectivec
- (IBAction)changeColor
{
    //create a keyframe animation
    CAKeyframeAnimation *animation = [CAKeyframeAnimation animation];
    animation.keyPath = @"backgroundColor";
    animation.duration = 2.0;
    animation.values = @[
                         (__bridge id)[UIColor blueColor].CGColor,
                         (__bridge id)[UIColor redColor].CGColor,
                         (__bridge id)[UIColor greenColor].CGColor,
                         (__bridge id)[UIColor blueColor].CGColor ];
    //add timing function
    CAMediaTimingFunction *fn = [CAMediaTimingFunction functionWithName: kCAMediaTimingFunctionEaseIn];
    animation.timingFunctions = @[fn, fn, fn];
    //apply animation to layer
    [self.colorLayer addAnimation:animation forKey:nil];
}
```

### 自定义缓冲动画

