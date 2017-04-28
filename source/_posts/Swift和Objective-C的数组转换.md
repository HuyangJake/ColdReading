---
title: Swift和Objective-C的数组转换
date: 2017-01-20 10:45:57
tags: [iOS, Swift]
---

1. 当我们在 Objective-C 代码中使用 Swift 类或者协议时，接入的API会将全部所有类型的Swift数组代替为NSArray。
2. Swift 的编译器会在接入 Objective-C APIs 的时候将NSArray类替换成AnyObject[]
3. 若我们将一个NSArray对象传递给Swift的API并要求数组元素为一个新的类型，运行时就会产生错误。
4. 如果 Swift API 返回一个不能被转换为NSArray类型的 Swift 数组，错误也会随之产生。

>如果某个对象是 Objective-C 或者 Swift 类的实例，或者这个对象可以转换成另一种类型，那么这个对象则属于AnyObject类型的对象  _例如，一个Int[]类型的 Swift 数组包含Int结构的元素。Int类型并不是一个类的实例，但由于Int类型转换成了NSNumber类，Int类型属于AnyObject类型的。_

<!-- more -->
### __Swift数组转换成NSArray__:

* 当我们从 Swift 数组转换为NSArray对象时，Swift 数组里的元素必须是属于AnyObject的。

* 当我们从一个 Swift 数组转换成一个NSArray对象后，转换后的结果则是一个[ObjectType]类型的数组.


### __NSArray对象转换成Swift数组__:

* 我们可以将任意NSArray对象转换成一个Swift数组，因为所有Objective-C的对象都是AnyObject类型。
* 如果NSArray对象没有指明一个确切的参数类型，那么它将会转换成[AnyObject]类型的Swift数组。

``` objectivec
//Objective
@property NSArray<NSDate *>* dates;
//Swift
var dates: [NSDate]

```
NSArray对象转换成一个 Swift 数组后，也可以将数组强制类型转换成一个特定的类型。，从AnyObject类型的对象转换成明确的类型并不会保证成功，从AnyObject[]转换为SomeType[]会返回一个 optional 的值。
例子：

``` swift
let swiftyArray = foundationArray as AnyObject[]
if let downcastedSwiftArray = swiftArray as? UIView[] {
    // downcastedSwiftArray contains only UIView objects
}
```

也可以在for循环中将NSArray对象定向地强制转换为特定类型的Swift数组:

``` swift
for aView: UIView! in foundationArray {
     // aView is of type UIView
}
```

参考：[Using Swift with Cocoa and Objective-C](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/index.html)