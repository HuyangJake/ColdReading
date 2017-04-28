---
title: JSPatch遇上Swift
date: 2017-02-08 14:42:27
tags: [iOS, Swift]
---

JSPatch的动态更新是依赖于Objective-C的runtime，那作为静态语言的Swift就没有办法使用JSPatch了吗？Swift类成员还是可以使用Objective-C的运行时动态派发，只要Swift类是继承自`NSObject`或者使用`dynamic`修饰的类的成员。

### 纯Swift类没有动态性 无法重写纯swift类的方法和属性
JSPatch在进行到overrideMethod进行方法实现IMP替换时要求class实现NSCoping协议，而不继承自NSObject的swift类是不遵循该协议的，因此会崩溃。

<!-- more -->

### Swift中使用Method Swizzling的原则
_摘自底部参考资料_

1. 继承自NSObject的Swift类，其继承自父类的方法具有动态性，其他自定义方法、属性需要加dynamic修饰才可以获得动态性。
2. 若方法的参数、属性类型为Swift特有、无法映射到Objective-C的类型(如Character、Tuple)，则此方法、属性无法添加dynamic修饰（会编译错误）。
3. 纯Swift类没有动态性，但在方法、属性前添加dynamic修饰可以获得动态性。

### 官方文档dynamic声明修饰符释义

> 该修饰符用于修饰任何兼容 Objective-C 的类的成员。访问被dynamic修饰符标记的类成员将总是由OC运行时系统进行动态派发，而不会由编译器进行内联或消虚拟化。


### JSPatch上手使用

* 继承自`NSObject`的Swift类中

``` swift
class ViewController: UIViewController {
	override func viewWillAppear(_ animated: Bool) {
         super.viewWillAppear(animated)
        self.view.backgroundColor = UIColor.yellow;
    }
}
```

重载自父类的方法不需要dynamic修饰符进行修饰，可以直接进行JSPatch动态替换。


* 自定义的Swift方法需要加上dynamic修饰符

``` swift
class ViewController: UIViewController {
	override func viewWillAppear(_ animated: Bool) {
         super.viewWillAppear(animated)
        self.view.backgroundColor = UIColor.yellow;
    }
}

dynamic func test() {
         print("这是原生的打印")
    }

```

* Swift中Cocoa的API方法跟Objective-C不同时，JSPatch代码

``` swift
 	dynamic func paramTest(paramOne: String,  paramTwo: String) -> String {
        return "原生的字符串";
    }
    
    dynamic func paramTest2(_ paramOne: String, withParam paramTwo: String) -> String {
        return "原生字符串2"
    }
    
    dynamic func paramTest3(withParamOne paramOne: String, andParamTwo paramTwo: String) -> String {
        return "原生字符串3"
    }

```

``` swift
 paramTestWithParamOne_paramTwo: function(paramOne, paramTwo) {
                return "JS字符串"
            },
            
 paramTest2_withParam: function(paramOne, paramTwo) {
                return "JS字符串2"
            },
            
 paramTest3WithParamOne_andParamTwo: function(paramOne, paramTwo) {
                return "JS字符串3"
            },
```

* defineClass中指定类名需要带上项目target的名字

``` swift
require('UIColor')
defineClass("Swift_JSPatch.ViewController", {
            
            viewWillAppear: function(animated) {
                self.super().viewWillAppear(animated)
                self.view().setBackgroundColor(UIColor.redColor())
            },
            
            test: function() {
                console.log("JS输出的代码")
            },
            
            paramTestWithParamOne_paramTwo: function(paramOne, paramTwo) {
                return "JS字符串"
            },
            
            paramTest2_withParam: function(paramOne, paramTwo) {
                return "JS字符串2"
            },
            
            paramTest3WithParamOne_andParamTwo: function(paramOne, paramTwo) {
                return "JS字符串3"
            },
            
            }, {})

```


参考：[JSPatch在Swift中的应用](http://www.jianshu.com/p/e2eb7b4861c5)

