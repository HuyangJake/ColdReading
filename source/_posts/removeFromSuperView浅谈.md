---
title: removeFromSuperView浅谈
date: 2017-03-29 17:26:16
tags: iOS
categories: [猿猿养成记]
---
>Unlinks the receiver from its superview and its window, and removes it from the responder chain.

译：把当前View从它的父View和窗口中移除，同时也把它从响应事件操作的响应者链中移除。

###  方法调用后的内存管理

先来看一段代码：
<!--more-->

``` objectivec
UIView *view = [UIView new];
self.testView = view;
[self.view addSubview:view];
self.title = @"jake";
[self.testView removeFromSuperview];

@try {
        for (int i = 0; i < 10; i ++) {
            NSLog(@"前 %ld", CFGetRetainCount((__bridge CFTypeRef)self.testView));
            [self.testView removeFromSuperview];
            NSLog(@"后 %ld", CFGetRetainCount((__bridge CFTypeRef)self.testView));
            self.testView.backgroundColor = [UIColor redColor];
            
            
            [self.testView setUserInteractionEnabled:YES];
            [self.testView removeFromSuperview];
            [self.testView setNeedsDisplay];
        }
    } @catch (NSException *exception) {
        NSLog(@"name : %@, reason : %@", exception.name, exception.reason);
    } @finally {
        
    }
```

这段代码执行不会崩溃，`self.testView`执行了多次`removeFromSuperView`也没有问题。

苹果爸爸对这个方法这样说的：
>The view is also released; if you plan to reuse it, be sure to retain it before sending this message and to release it as appropriate when adding it as a subview of another NSView.

调了这个方法就release了这个view，叫我们想要再次使用就得把它retain下来。

---
结果我执行上段代码，打印出来`testView`的`retainCount`不但没有减少反而增加了1  (;￢＿￢)   ，不过在调用完（这个runloop结束）之后还是会release的。好吧，其实reatainCount不能太当回事，_之前在MRC时期经常发现retainCount不准确，这主要是因为iOS系统API的引用、或自动释放池导致的，所以retainCount并不能当做可靠的参考。_

---
__无论是ARC还是MRC中多次调用removeFromSuperview和addSubview:方法，都不会造成造成重复释放和添加。__

__永远不要在你的View的drawRect:方法中调用removeFromSuperview。__
 


