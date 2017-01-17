---
title: Objective-C中的装饰模式
date: 2017-01-10 14:34:56
tags: iOS
---

### 背景
前段时间接触到了装饰模式，也做了基本的了解，但是还不是很清楚它在iOS开发中的实际运用，和合理的存在方式。这两天看了__《Objective-C编程之道：iOS设计模式解析》__ 中的第16章，有了更加深的理解，结合自己的理解在这里做一下记录。

### 什么是装饰模式
就描述概念而言，我觉得__《Head First 设计模式》__这本书通过各种例子阐述得要更加详细易懂。__《Objective-C编程之道：iOS设计模式解析》__讲得更多是iOS开发中的使用和不同实现方式的对比。

标准的装饰模式有包括一个抽象的Component父类，它声明了一些操作，它具体的类讲进行重载以实现自己特定的操作。这个Component具体类是模式中的被装饰者，Component父类可以被细化为另一个叫做Decorator的抽象类，即装饰者抽象类。Decorator类中包含了一个Component的引用。Decorator的具体类为Component或者Decorator定义了几个扩展行为，并且会在自己的操作中内嵌Component操作。关系图见 [__装饰模式类图__](#class map)

Component定义了一些抽象操作，具体类将进行重载实现自己特定的操作。Decorator抽象类通过将一个Component（或Decorator）内嵌到Decorator对象，定义了扩展这个Component的实例的“装饰性”的行为。

默认的operation方法只是想内嵌的Component发送一个消息，Decorator的具体实现类重载父类的operation，通过super把自己增加的行为扩展给Component的operation。如果只需要向Component添加一种职责，那可以省掉抽象的Decorator类，让具体的Decorator直接把请求转发给Component。那么这种方式就好像形成一种操作链，把一种行为加到另一种行为之上，如[__对象图__](#class map)

#### <span id="class map">装饰模式类图</span>
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/%E8%A3%85%E9%A5%B0%E6%A8%A1%E5%BC%8F%E7%B1%BB%E5%9B%BE.png)

### 何时使用装饰模式
* 想要在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责。

* 想要扩展一个类的行为，却做不到。类定义可能被隐藏，无法进行子类化；或者对类的每个行为的扩展，为支持每种功能组合，将产生大量的子类

* 对类的职责的扩展是可选的。

### 装饰模式在iOS中的实现
根据Objective-C的特性，有两种实现方式：

1. [通过真正的子类实现装饰](#child)
2. [通过分类实现装饰](#category)

第二种方式是使用了Objective-C的语言功能，通过分类向类添加行为，不必进行子类化，这并非标准的装饰模式结构，但是实现了装饰模式同样的需求。尽管使用分类来实现装饰模式跟原始风格有偏离，但是实现少量的装饰器的时候，它比真正子类方式更加轻量、更加容易。

例子是使用装饰实现UIImage的滤镜功能 终于要见代码了~

---
#### <span id="child">真子类实现装饰</span>
先看目录结构：
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/%E7%9C%9F%E6%AD%A3%E5%AD%90%E7%B1%BB%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.png)

Component抽象类是一个Protocol文件，

``` objectivec

@protocol ImageComponent <NSObject>
@optional
- (void)drawAsPatternInRect:(CGRect)rect;
- (void)drawAtPoint:(CGPoint)point;
- (void)drawAtPoint:(CGPoint)ponit blendMode:(CGBlendMode)blendMode alpha:(CGFloat)alpha;
- (void)drawInRect:(CGRect)rect;
- (void)drawInRect:(CGRect)rect blendMode:(CGBlendMode)blendMode alpha:(CGFloat)alpha;
@end

```

`UIImage_ImageComponent.h`是UIImage的一个Extension，让UIImage遵循ImageComponent协议。如此在装饰模式中UIImage类是ImageComponent抽象类的具体类，即UIImage是模式中的被装饰者。

Decorator抽象类`ImageFilter`遵守ImageComponent协议，并定义了三个方法和一个ImageComponent协议的代理对象属性。

``` objectivec

@interface ImageFilter : NSObject <ImageComponent>
- (void)apply;
- (id)initWithImageComponent:(id <ImageComponent>) component;
- (id)forwardingTargetForSelector:(SEL)aSelector;
- 
@end

```

``` objectivec

@interface ImageFilter ()
@property (nonatomic, strong) id <ImageComponent> component;
@end
@implementation ImageFilter

- (instancetype)initWithImageComponent:(id<ImageComponent>)component {
    if (self = [super init]) {
        [self setComponent:component];
    }
    return self;
}
- (void)apply {
    //由子类重载
}
- (id)forwardingTargetForSelector:(SEL)aSelector {
    NSString *selectorName = NSStringFromSelector(aSelector);
    if ([selectorName hasPrefix:@"draw"]) {
        [self apply];
    }
    return self.component;
}

@end

```

在Decorator抽象类的具体实现类中去重载apply方法，当发送以"draw"开头的消息时，先执行Decorator具体类中的apply方法，然后再执行Component具体类即UIImage类（被装饰者）中的"draw"开头的那个方法。如此实现了对UIImage对应的"draw"开头方法动态添加行为。

具体的使用代码如下：

``` objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
    UIImage *image = [UIImage imageNamed:@"avatar.jpg"];
    CGAffineTransform rorateTransform = CGAffineTransformMakeRotation(-M_PI / 4);
    CGAffineTransform translateTransform = CGAffineTransformMakeTranslation(-image.size.width / 2.0, image.size.height / 8.0);
    CGAffineTransform finalTransform = CGAffineTransformConcat(rorateTransform, translateTransform);
    
    id<ImageComponent>transformdImage = [[ImageTransformFilter alloc] initWithImageComponent:image transform:finalTransform];
    
    id<ImageComponent>finalImage = [[ImageShadowFliter alloc] initWithImageComponent:transformdImage];
    
    DecoratorView *decoratorView = [[DecoratorView alloc] initWithFrame:self.view.frame];
    [decoratorView setImage:finalImage];
    [self.view addSubview:decoratorView];
}

```
由`DecoratorView`中的`drawRect：`调起UIImage的装饰

``` objectivec
@implementation DecoratorView
during animation.
- (void)drawRect:(CGRect)rect {
    [self.image drawInRect:rect];
}
@end

```

__类图如下:__
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/%E6%AD%A3%E7%9C%9F%E5%AD%90%E7%B1%BB%E7%B1%BB%E5%9B%BE.png)

各种ImageComponent对象可在运行时进行连接，如下图：
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/ImageComponent%E9%93%BE%E6%8E%A5.png)

---
#### <span id="category">分类实现装饰</span>

分类方式实现就是平常开发中常见的使用，代码就不再一一列出

类图如下：
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/%E5%88%86%E7%B1%BB%E7%B1%BB%E5%9B%BE.png)

`BaseFilter`分类中定义了绘图的方法，`Transform`和`shadow`分类中可以调用`BaseFilter`分类中定义的方法来进行自己的绘制。他们也可以像真正子类化方式那样链接起来，来看对象图:
![](http://ojam5z7vg.bkt.clouddn.com/coldreading/jpg/%E5%88%86%E7%B1%BB%E9%93%BE%E6%8E%A5%E5%AF%B9%E8%B1%A1%E5%9B%BE.png)

---
### 总结
装饰模式在Objective-C中有两种不同的实现方式，真正子类方式的实现使用一种较为结构化的方式链接各种装饰器，分类的方式更加简单和轻量，使用于现有类只需要少量装饰器的应用。

参考：《Objective-C编程之道：iOS设计模式解析》第16章，（190-206）
demo地址：[GitHub](https://github.com/HuyangJake/DecoratorDemo.git)
