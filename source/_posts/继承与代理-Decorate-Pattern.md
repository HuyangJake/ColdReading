---
title: 继承与代理--Decorate Pattern
date: 2017-01-04 17:51:11
tags: iOS
---

### 场景
写这篇文章的背景是在看[casatwy的网络层架构代码](https://github.com/casatwy/RTNetworking)时对子类继承和协议代理方的实现产生了疑惑，进行了探索。

### 问题
问题情景：父类的方法列表中和协议中有同样的方法（_代码如下_），子类继承方法同时代理方也实现协议方法，那么当父类调用`decoratePatternTest`这个方法时具体的执行方是谁？子类同时是代理方和不是代理方分别有什么样的情况呢？

<!-- more -->

``` objectivec

#import <Foundation/Foundation.h>
@protocol BaseDecorateDelegate <NSObject>
- (NSString *)decoratePatternTest;
@end
@interface BaseProtocolTest : NSObject
@property (nonatomic, weak) id<BaseDecorateDelegate> decorate;
- (NSString *)decoratePatternTest;
- (void)startTest;
@end

```

### casatwy注释

在casatwy的代码中遇到上述问题的有`BaseAPIManager`拦截器代码部分，作者的注释如下：

``` objectivec
/*
    拦截器的功能可以由子类通过继承实现，也可以由其它对象实现,两种做法可以共存
    当两种情况共存的时候，子类重载的方法一定要调用一下super
    然后它们的调用顺序是BaseManager会先调用子类重载的实现，再调用外部interceptor的实现
    
    notes:
        正常情况下，拦截器是通过代理的方式实现的，因此可以不需要以下这些代码
        但是为了将来拓展方便，如果在调用拦截器之前manager又希望自己能够先做一些事情，所以这些方法还是需要能够被继承重载的
        所有重载的方法，都要调用一下super,这样才能保证外部interceptor能够被调到
        这就是decorate pattern
 */
 
```

注释中提到：

* 子类继承和代理方实现协议可以共存
* 两种共存时子类重载的方法必须要调用父类方法
* 两者调用顺序是先实现子类重载的方法再调用代理方实现的方法
* 最后提到的是 __Decorate Pattern__ 装饰者模式

### 分情况实现验证

1. 子类继承父类方法 & 子类没有遵守父类协议 & 其他类遵守协议实现代理方法
2. 子类继承父类方法 & 遵守父类协议 

父类方法：

``` objectivec

- (NSString *)decoratePatternTest {
    NSLog(@"执行父类decoratePatternTest");
    NSString *string = @"基类中实现decoratePatternTest";
    if (self != self.decorate && [self.decorate respondsToSelector:@selector(decoratePatternTest)]) {
       string = [self.decorate decoratePatternTest];
    }
    return string;
}

```

子类方法：

``` objectivec

- (NSString *)decoratePatternTest {
    [super decoratePatternTest];
    NSLog(@"子类执行decoratePatternTest");
    return @"子类中实现decoratePatternTest";
}

```

非子类代理者方法：

``` objectivec

- (NSString *)decoratePatternTest {
    NSLog(@"执行过代理方实现decoratePatternTest");
    return @"代理方实现decoratePatternTest";
}

```

测试结果：


||父类|子类|Other||父类|子类|Other|
|-|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|遵守协议实现方法|-|N|Y||-|Y|N|
|继承方法|-|Y|N||-|Y|N|
|方法执行顺序|2|1|3||2|1|-|

### 关键

其实问题的关键是在代理者的协议方法调用的时机。

![](http://ojam5z7vg.bkt.clouddn.com/clodreading/jpg/%E7%BB%A7%E6%89%BF%E4%B8%8E%E4%BB%A3%E7%90%86.png-blogwebp)

* 当子类继承了方法又是代理者，那么在父类方法中通过`（self != self.decotate）`来判断不用再去执行协议中的方法

* 子类不是代理者没有继承方法，则在父类方法中调用代理者执行协议方法

* 子类不是代理者但是继承了方法，父类调用代理者协议方法
	* 两个方法的调其顺序是 子类中方法 -> 父类中方法 -> 代理者方法
	* 两个方法中的有效代码执行顺序要看 子类方法中 `super`调用的位置 (`super`调用在子类有效代码前则 代理者方法的代码先执行)------->见最后一节__Decorate Pattern__ 装饰者模式
	
#### 代码中不用调用父类方法的情况
作者的代码`BaseAPIManager`中的`reformerParams`方法在父类实例方法和协议方法中存在，但是子类继承实现时并不需要调用父类的 `reformerParams`，原因是父类中作了如下处理：
 
``` objectivec

- (instancetype)init
{
    self = [super init];
    if (self) {
        if ([self conformsToProtocol:@protocol(BaseProtocolDelegate)]) {
            self.child = (id <BaseProtocolDelegate>)self;
        } else {
            NSException *exception = [[NSException alloc] init];
            @throw exception;
        }

    }
    return self;
}

- (NSString *)reformerParams {
    NSLog(@"执行父类reformerTest");
    IMP childIMP = [self.child methodForSelector:@selector(reformerTest)];
    IMP selfIMP = [self methodForSelector:@selector(reformerTest)];
    
    if (childIMP == selfIMP) {
        return @"子类没有继承此方法";
    } else {
        // 如果child是继承得来的，那么这里就不会跑到，会直接跑子类中的IMP。
        // 如果child是另一个对象，就会跑到这里
        NSString *result = nil;
        result = [self.child reformerParams];
        if (result) {
            return result;
        } else {
            return @"default";
        }
    }
}
```

* 初始化方法中首先保证子类必须遵守了`BaseProtocolDelegate`协议

* 子类实现了`reformerParams`方法并没有调用`super`并不会出发父类的`reformerParams`

* 根据消息传递机制父类`reformerParams` 方法会被调用说明子类没有实现此方法，那么执行非子类代理者的协议方法

* 父类要求子类必须遵守协议，那么此协议`BaseProtocolDelegate`更多的用处是获取子类对象

### __Decorate Pattern__ 装饰者模式

>__装饰者模式__动态地将责任附件到对象上，若要扩展功能，装饰着提供了比继承更具有弹性的方案。

#### 装饰者模式的设计原则：
![](http://ojam5z7vg.bkt.clouddn.com/clodreading/jpg/%E8%A3%85%E9%A5%B0%E8%80%85%E6%A8%A1%E5%BC%8F%E8%AE%BE%E8%AE%A1%E5%8E%9F%E5%88%99.png-blogwebp)
可以随心为一个类扩展功能，但不允许对已经存在的代码进行修改。

#### 装饰者模式的主要特点：
![](http://ojam5z7vg.bkt.clouddn.com/clodreading/jpg/%E8%A3%85%E9%A5%B0%E8%80%85%E6%A8%A1%E5%BC%8F%E7%89%B9%E7%82%B9.png-blogwebp)

#### 装饰者模式类图：
![](http://ojam5z7vg.bkt.clouddn.com/clodreading/jpg/%E8%A3%85%E9%A5%B0%E8%80%85%E6%A8%A1%E5%BC%8F%E7%B1%BB%E7%A4%BA%E6%84%8F%E5%9B%BE.png-blogwebp)

#### 实际的例子：java中常用的java.io类就存在着大量装饰者
![](http://ojam5z7vg.bkt.clouddn.com/clodreading/jpg/%E8%A3%85%E9%A5%B0%E8%80%85java.io.png-blogwebp)

以上内容摘自《Head First设计模式》 第三章 装饰者模式 （79-107）

对应到iOS中本文的场景：


|被装饰者|功能组件|装饰者|
|:-:|:-:|:-:|
|委托方父类|代理方OtherObject（非子类）|子类|

其实在思考装饰者模式对应情况上，有点疑惑： 装饰者类Decorator没有找到对应的类

但装饰者模式中的关键点：

>装饰者可以在被所委托被装饰者的行为之前或之后，加上自己的行为，以达到特定目标。

在本文的例子中有明显体现：子类继承的方法中调用`super`方法，在这之前或者之后可以加上自己的行为，达到特定目标。

对装饰者模式的理解还比较基础有待继续研究……欢迎指正指导，谢谢！

iOS中装饰者模式的实现的后续研究请看[Objective-C中的装饰模式](/2017/01/10/Objective-C中的装饰模式/index.html)
 
