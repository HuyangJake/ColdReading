---
title: Using Swift with Cocoa and Objective-C笔记_ONE
date: 2017-01-19 15:49:18
tags: iOS Swift
---

## Objective-C和Swift的非空值、可选值转换
Objective-C 能够使用空值标记来设定一个参数类型，属性类型或者返回值类型是否可以为 NULL 或者 为 nil 值。单独的类型声明可以使用__nullable和__nonnull标注，空值的范围性的声明可以使用NS_ASSUME_NONNULL_BEGIN和NS_ASSUME_NONNULL_END宏。如果一个类型没有任何的空值标注信息，Swift 就不能分辨出可选值和非可选值类型，并且将作为隐式的解包可选值导入。

* 以__nonnull或者范围宏标注声明的空值类型，被作为非空可选值non-optional导入到 Swift。
* 以__nullable标注声明的空值类型，被作为可选值导入到 Swift。
* 没有以空值标注声明的类型被作为隐式的解包可选值导入到 Swift。

<!-- more -->
Objective-C申明

``` objectivec
@property (nullable) id  nullableProperty;
@property (nonnull) id nonNullProperty;
@property id unannotatedProperty;
NS_ASSUME_NONNULL_BEGIN
- (id)returnsNonNullValue;
- (void)takesNonNullParameter:(id)value;
NS_ASSUME_NONNULL_END
- (nullable id)returnsNullableValue;
- (void)takesNullableParameter:(nullable id)value;
- (id)returnsUnannotatedValue;
- (void)takesUnannotatedParameter:(id)value;
```

导入Swift中：

``` swift
var nullableProperty: AnyObject?
var nonNullProperty: AnyObject
var unannotatedProperty: AnyObject!
func returnsNonNullValue() -> AnyObject
func takesNonNullParameter(value: AnyObject)
func returnsNullableValue() -> AnyObject?
func takesNullableParameter(value: AnyObject?)
func returnsUnannotatedValue() -> AnyObject!
func takesUnannotatedParameter(value: AnyObject!)
```


### 闭包和Block的交互

Objective-C 中的blocks会被自动导入为 Swift 中的闭包。例如，下面是一个 Objective-C 中的 block 变量：
 
``` objectivec
void (^completionBlock)(NSData *, NSError *) = ^(NSData *data, NSError *error) {/* ... */}
```

而它在 Swift 中的形式为:

``` swift
let completionBlock: (NSData, NSError) -> Void = {data, error in /* ... */}
```

Swift 的闭包与 Objective-C 中的 blocks 能够和睦相处，所以你可以把一个 Swift 闭包传递给一个把 block 作为参数的 Objective-C 函数。Swift 闭包与函数具有互通的类型，所以你甚至可以传递 Swift 函数的名字。

闭包与 blocks 语义上相通但是在一个地方不同：变量是可以直接改变的，而不是像 block 那样会拷贝变量。换句话说，Swift 中变量的默认行为与 Objective-C 中 __block 变量一致。


### Swift类型兼容性 @objc 和 dynamic

@objc可以让你的 Swift API 在 Objective-C 中使用。换句话说，你可以通过在任何 Swift 方法、类、属性前添加@objc，来使得他们可以在 Objective-C 代码中使用。如果你正在使用如键值观察的 API 来动态替换方法的实现，也可以通过使用dynamic修饰符来获得对 Objective-C 运行时被自动派发的成员的访问。

当你在 Objective-C 中使用 Swift API，编译器通常会对语句做直接的翻译。例如，Swift API func playSong(name: String)会被解释为`- (void)playSong:(NSString *)name`。然而，有一个例外：当在 Objective-C 中使用 Swift 的初始化函数，编译器会在方法前添加“initWith”并且将原初始化函数的第一个参数首字母大写。例如，这个 Swift 初始化函数`init (songName: String, artist: String)`将被翻译为`- (instancetype)initWithSongName:(NSString *)songName artist:(NSString *)artist `。

Swift 同时也提供了一个@objc关键字的变体，通过它你可以自定义在 Objective-C 中转换的函数名。

例子：

``` swift
//Swift
@objc(Squirrel)
class Белка {
    @objc(initWithName:)
    init (имя: String) { /*...*/ }
    @objc(hideNuts:inTree:)
    func прячьОрехи(Int, вДереве: Дерево) { /*...*/ }
}
```



### 轻量级泛型

``` objectivec
@property NSArray<NSDate *>* dates;
@property NSSet<NSString *>* words;
@property NSDictionary<KeyType: NSURL *, NSData *>* cachedData;
```

``` swift
var dates: [NSDate]
var words: Set<String>
var cachedData: [NSURL: NSData]
```

>注意 除了 Foundation 中的集合类， Objective-C 的轻量级泛型会被 Swift 忽略掉。任何其他使用轻量级泛型的类型在导入到 Swift 中时会被视为无参数化。

---