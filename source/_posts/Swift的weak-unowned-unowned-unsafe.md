---
title: Swift的weak unowned unowned(unsafe)
date: 2017-01-19 11:56:19
tags: iOS Swift
---

Swift同样也是用ARC管理内存，为了解决引用循环。同OC一样使用weak修饰对象，在weak引用指向的对象被释放之后，其自身会被置为nil也会被释放。

而在Swift中nil是Optional的专属，对于非可选类型，则使用无主引用：unowned

>使用无主引用，你必须确保引用始终指向一个未销毁的实例。 
如果你试图在实例被销毁后，访问该实例的无主引用，会触发运行时错误。

* 安全无主引用`unowned`指向的对象被释放之后，其自身不会被置为nil，也不可访问。

* 不安全无主引用`unowned（unsafe）`指向的对象被释放之后，你试图去访问这个引用，程序会尝试访问该实例之前坐在的内存地址，__这是一个不安全的操作__
<!-- more -->
#### 使用场景解析：
前提描述： A类中有一个B类的属性， B类中有一个A类的属性

1. 两个属性的值都允许为nil，并会潜在产生循环引用
	- 这种情况适合使用weak来解决
	 
2. 其中一个属性的值允许为nil，而另一个不允许为nil
	- 这种情况最适合通过无主引用来解决
	
3. 两个属性都必须有值，并且初始化完成后永远不会为nil
	- 这种情况，需要一个雷使用无主属性，为另外一个类使用隐式解析可选属性
	
场景3例子：
	
``` swift
class Country { 
	let name: String 
	var capitalCity: City! 
	init(name: String, capitalName: String) { 
		self.name = name 
		self.capitalCity = City(name: capitalName, country: self) 
	} 
}

class City { 
	let name: String 
	unowned let country: Country 
	init(name: String, country: Country) { 
		self.name = name 
		self.country = country 
	} 
}
```

---

### 解决闭包引起的循环引用

>Swift 有如下要求：只要在闭包内使用 self 的成员，就要用 self.someProperty 或者 self.someMethod() 不只是 someProperty 或 someMethod() ）。这提醒你可能会一不小心就捕获了 self 。

第一捕获列表解决循环引用，例子：

捕获列表中的每一项都由一对元素组成，一个元素是 weak 或 unowned 关键字，另一个元素是类实例的引用

``` swift
lazy var someClosure: (Int, String) -> String = { 
	[unowned self, weak delegate = self.delegate!] (index: Int, stringToProcess: String) -> String in 
	// 这里是闭包的函数体 
}
```

>如果被捕获的引用绝对不会变为 nil ，应该用无主引用，而不是弱引用。

---
