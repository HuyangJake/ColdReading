---
title: SwiftUI 数据流 
date: 2020-10-09 10:14:39
tags: [SwiftUI]
---


>“纯函数指的是，返回值只由调用时的参数决定，而不依赖于任何系统状态，也不改变其作用域之外的变量状态的函数”

>摘录来自: 王 巍. “SwiftUI 和 Combine 编程。” Apple Books. 
>

<!-- more -->

使用纯函数的方式可以和SwiftUI配合得天一无缝，SwiftUI强调single source of truth, 一个稳定可测试的model是app状态清晰的最重要的保证，也是SwiftUI中传递状态时首先需要考虑的方向。

### 前端流行的框架

Redux 为代表的状态管理和组件通讯架构思想和步骤：

>
1. 将 app 当作一个状态机，状态决定用户界面。
1. 这些状态都保存在一个 Store 对象中，被称为 State。
2. View 不能直接操作 State，而只能通过发送 Action 的方式，间接改变存储在 Store 中的 State。
3. Reducer 接受原有的 State 和发送过来的 Action，生成新的 State。
4. 用新的 State 替换 Store 中原有的状态，并用新状态来驱动更新界面。


![Redux架构](http://qiniu.huyangjie.cn/mweb/15981578385918.png)


在小型的项目里面不一定需要引入Redux这样的架构，这样反而会增加复杂性。

## “@ ” 属性包装
在Swift 中 @ 属性的特性称为 属性包装， propertyWrapper

## @State

@State 修饰的值，在SwiftUI内部会被自动转换为一对 setter 和 getter， 对这个属性进行赋值的操作将会触发View的刷新，它的body会被再次调用，低层渲染引擎会找出界面上被改变的部分，根据新的属性值计算出新的View 并进行刷新。

## @Binding

@State 属性值仅只能在属性本身被设置时会触发UI刷新，这个特性让它非常适合用来声明一个值类型的值：因为对值类型的属性的变更，也会触发整个值的重新设置，进而刷新 UI。

不过，在把这样的值在不同对象间传递时，状态值将会遵守值语义发生复制。如此各个层级的的State属性都不相同，对State属性的变更就只能作用在同层级中，对顶层的UI就没有办法做出更新。

_@Binding_ 就是解决这个问题。

>“@Binding 也是对属性的修饰，它做的事情是将值语义的属性“转换”为引用语义。对被声明为 @Binding 的属性进行赋值，改变的将不是属性本身，而是它的引用，这个改变将被向外传递。”

>摘录来自: 王 巍. “SwiftUI 和 Combine 编程。” Apple Books. 
>

在 Swift 5.1 中，对一个由 @ 符号修饰的属性，在它前面使用 $ 所取得的值，被称为投影属性 (projection property)。有些 @ 属性，比如 @State 和 @Binding，它们的投影属性就是自身所对应值的 Binding 类型。

``` swift
CalculatorButtonRow(row: row, brain: self.$brain)
```

$brain 的写法将 brain 从 State 转换成了引用语义的 Binding

### 状态传递选择

1. 这个状态是属于单个 View 及其子层级，还是需要在平行的部件之间传递和使用？@State 可以依靠 SwiftUI 框架完成 View 的自动订阅和刷新，但这是有条件的：对于 @State 修饰的属性的访问，只能发生在 body 或者 body 所调用的方法中。你不能在外部改变 @State 的值，它的所有相关操作和状态改变都应该是和当前 View 挂钩的。如果你需要在多个 View 中共享数据，@State 可能不是很好的选择；如果还需要在 View 外部操作数据，那么 @State 甚至就不是可选项了。

2. 状态对应的数据结构是否足够简单？对于像是单个的 Bool 或者 String，@State 可以迅速对应。含有少数几个成员变量的值类型，也许使用 @State 也还不错。但是对于更复杂的情况，例如含有很多属性和方法的类型，可能其中只有很少几个属性需要触发 UI 更新，也可能各个属性之间彼此有关联，那么我们应该选择引用类型和更灵活的可自定义方式。


## ObservableObject 和 @ObjectBinding

ObservableObject 协议要求实现类型是class， 它只有一个需要实现的属性：objectWillChange, 在数据将要发生改变时，这个属性用来向外进行“广播”，它的订阅者（一般是View相关的逻辑）收到通知后，对View进行刷新。

创建好ObservableObject后在View中使用的时候需要将它声明为@ObservedObject，它也是一个属相包装，负责通过订阅 objectWillChange 将具体管理数据的 ObservableObject 和 当前的View关联起来。

🌰
``` swift
class CalculatorModel: ObservableObject {
    let objectWillChange = PassthroughSubject<Void, Never>()
    var brain: CalculatorBrain = .left("0") {
        willSet { objectWillChange.send() }
    }
}
```

## @Published

当ObservableObject中存在大量需要手动调用objectWillChange.send(),使用@Published ,编译器将会帮我们自动完成这件事情。

``` swift
class CalculatorModel: ObservableObject {
    @Published var brain: CalculatorBrain = .left("0")
}
```

## @EnvironmentObject

>“在 SwiftUI 中，View 提供了 environmentObject(_:) 方法，来把某个 ObservableObject 的值注入到当前 View 层级及其子层级中去。在这个 View 的子层级中，可以使用 @EnvironmentObject 来直接获取这个绑定的环境值。”

>摘录来自: 王 巍. “SwiftUI 和 Combine 编程。” Apple Books. 
>

在SceneDelegate.swift文件中创建ContentView的地方，通过environmentObject把通用的 Model 添加上去：

``` swift
window.rootViewController = UIHostingController(
    rootView: ContentView().environmentObject(CalculatorModel())
)
```

为了让 ContentView 的预览也能保持工作，你还需要为在ContentView_Previews 中也用同样的方式添加上 environmentObject：

``` swift
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
    ContentView().environmentObject(CalculatorModel())
    }
}
```

## 小结

>“@State 和 @Binding 提供 View 内部的状态存储，它们应该是被标记为 private 的简单值类型，仅在内部使用。ObservableObject 和 @ObservedObject 则针对跨越 View 层级的状态共享，它可以处理更复杂的数据类型，其引用类型的特点，也让我们需要在数据变化时通过某种手段向外发送通知 (比如手动调用 objectWillChange.send() 或者使用 @Published)，来触发界面刷新。对于“跳跃式”跨越多个 View 层级的状态，@EnvironmentObject 能让我们更方便地使用 ObservableObject，以达到简化代码的目的。”

摘录来自: 王 巍. “SwiftUI 和 Combine 编程。” Apple Books. 


### Reference
文章代码以及知识梳理来自于喵神的《SwiftUI 和 Combine 编程》，感兴趣可以在objc.io上购买食用。