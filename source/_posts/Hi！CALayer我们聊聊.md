---
title: Hi！CALayer我们聊聊
date: 2017-07-17 15:39:59
tags: [iOS, 学习笔记]
---

CALayer和UIView最大的不同就是，CALayer不处理用户交互。因为它并不清楚具体的响应链。

### 1. iOS 和 MacOS 的坐标系统 

1.  __点__ —— 在iOS和Mac OS中最常见的坐标体系。点就像是虚拟的像素，也被称 作逻辑像素。在标准设备上，一个点就是一个像素，但是在Retina设备上，一 个点等于2*2个像素。iOS用点作为屏幕的坐标测算体系就是为了在Retina设备 和普通设备上能有一致的视觉效果。 

2. __像素__ —— 物理像素坐标并不会用来屏幕布局，但是仍然与图片有相对关系。 UIImage是一个屏幕分辨率解决方案，所以指定点来度量大小。但是一些底层 的图片表示如CGImage就会使用像素，所以你要清楚在Retina设备和普通设备 上，他们表现出来了不同的大小。 

3. __单位__ —— 对于与图片大小或是图层边界相关的显示，单位坐标是一个方便的 度量方式， 当大小改变的时候，也不需要再次调整。单位坐标在OpenGL这种 纹理坐标系统中用得很多，Core Animation中也用到了单位坐标。

<!--more-->

### 2. CALayer层的简单使用：

#### 2.1 给UIView设置寄宿图片

``` objectivec
//设置图片
layer.contents = (__bridge id)image.CGImage;

//设置图片的展示模式
layer.contentsGravity = kCAGravityResizeAspect;

```

`contentsGravity`的可选常量：

```
kCAGravityCenter 
kCAGravityTop 
kCAGravityBottom 
kCAGravityLeft 
kCAGravityRight 
kCAGravityTopLeft 
kCAGravityTopRight 
kCAGravityBottomLeft 
kCAGravityBottomRight 
kCAGravityResize 
kCAGravityResizeAspect 
kCAGravityResizeAspectFill
```

将`contentsScale`设置为1.0，将会以每个点1个像素绘制图片，如果设置为
2.0，则会以每个点2个像素绘制图片，这就是我们熟知的Retina屏幕。(前提是`contentsGravity`没有被设置)，视图层UIView有一个类似的属性`contentScaleFactor`

__当用代码的方式来处理寄宿图的时候，一定要记住要手动的设置图层 的 contentsScale 属性，否则，你的图片在Retina设备上就显示得不正确啦。__

``` objectivec
layer.contentsScale = [UIScreen mainScreen].scale;
```




#### 2.2 使用`contentsRect`实现image sprites（图片拼合）

主要思想：载入一张包含很多小图的大图，然后通过`contentsRect`将大图中的小图分别展示到不同的视图层中。

优点：图片的载入会更加快，提高了载入性能

contentsRect的`{0, 0, 0.5, 0.5}`效果
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/contentsRect.png-bigblog)

``` 

- (void)viewDidLoad 
{
  [super viewDidLoad]; //load sprite sheet
  UIImage *image = [UIImage imageNamed:@"Sprites.png"];
  //set igloo sprite
  [self addSpriteImage:image withContentRect:CGRectMake(0, 0, 0.5, 0.5) toLayer:self.iglooView.layer];
  //set cone sprite
  [self addSpriteImage:image withContentRect:CGRectMake(0.5, 0, 0.5, 0.5) toLayer:self.coneView.layer];
  //set anchor sprite
  [self addSpriteImage:image withContentRect:CGRectMake(0, 0.5, 0.5, 0.5) toLayer:self.anchorView.layer];
  //set spaceship sprite
  [self addSpriteImage:image withContentRect:CGRectMake(0.5, 0.5, 0.5, 0.5) toLayer:self.shipView.layer];
}

```

#### 2.3 使用`contentsCenter`调整可拉伸区域的大小
contentsCenter 其实是一个CGRect，它定义了一个固定的边框和一个在图 层上可拉伸的区域。

![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/contentsCenter.png-bigblog)

除此之外还可以用IB来控制`contentsCenter`属性。 在视图的第四个检查器中，有一个`stretching`属性，简直不要太赞👍


### Reference

[《iOS 核心动画》](https://zsisme.gitbooks.io/ios-/content/chapter2/the-contents-image.html)  感谢译者的付出！👍