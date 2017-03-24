---
title: Objective-C isa指针
date: 2016-12-19 13:52:37
tags: iOS 
---


### 1. 什么是isa指针

首先来看一下，NSObject的定义（不含方法定义）：

``` objectivec
 @interface NSObject <NSObject> {
     Class    isa;
 } 
 ```
 
　　在Objective-C中，@interface关键字可以看着是C语言中的struct关键字的别名，当然他还会有一些其它功能，比如说让编译器知道@interface后后面的是一个Objective-C的类的名字等。但就我们研究其内存布局来说，我们简单地将其替换为struct，并将protocal定义去掉。因此，NSObject的定义就是样：

``` objectivec
 struct NSObject{
 　　Class isa;
 }
 ```
 
那个这个Class又是什么呢？在objc.h中我们发现其仅仅是一个结构(struct)指针的typedef定义:
<!-- more -->

	 typedefstruct objc_class *Class;
 因此，NSObject的定义就像这个样子：

``` objectivec
 struct NSObject{
 　　objc_class *isa
 }
 ```

isa就是“is a”，对于所有继承了NSObject的类其对象也都有一个isa指针。这个isa指针指向的东西(先这样称呼它吧)就是关于这个对象所属的类的定义。

每一个Objective-C对象的底层都是这样的一个C结构体：

``` objectivec
struct objc_class {
  struct objc_class * isa;
  const char *name;
  ….
  struct objc_method_list **methodLists; /*方法链表*/
};
```

这个结构体中的第一个成员变量，就是isa指针。

### 2. isa指针的指向 
实例对象的isa指针指向是其所属的类对象，这个类对象包含了该实例对象的一些信息（例如：实例列表、方法列表等）。isa指针的类型是还是一个结构体 `objc_class` 其实也就是指向地址的类对象的结构，接下来看下 该`objc_class`的具体结构：


``` objectivec
struct objc_class {  
     Class isa;  
       
     Class super_class;  
       
     const char *name;  
       
     long version;  
     long info;  
       
     long instance_size;  
     struct objc_ivar_list *ivars;  
     struct objc_method_list **methodLists;   
       
     struct objc_cache *cache;  
     struct objc_protocol_list *protocols;     
 } 
```

一个objc_class对象包括一个类的：父类定义(super_class), 变量列表，方法列表，还有实现了哪些协议(Protocal)

这个结构中还有一个isa指针，它在这里（类对象中）指向的是元类对象(`metaclass object`)。在Objective-C中任何的类定义都是对象。即在程序启动的时候任何类定义都对应于一块内存。在编译的时候，编译器会给每一个类生成一个且只生成一个”描述其定义的对象”,也就是水果公司说的类对象(class object),他是一个单例(singleton), 而我们在C++等语言中所谓的对象，叫做实例对象(instance object)。对于实例对象我们不难理解，但类对象(class object)是干什么吃的呢？我们知道Objective-C是门很动态的语言，因此程序里的所有实例对象(instace objec)都是在运行时由Objective-C的运行时库生成的，而这个类对象(class object)就是运行时库用来创建实例对象(instance object)的依据。

__任何直接或间接继承了NSObject的类，它的实例对象(instacne objec)中都有一个isa指针，指向它的类对象(class object)。这个类对象(class object)中存储了关于这个实例对象(instace object)所属的类的定义的一切：包括变量，方法，遵守的协议等等。__

_ 这个实例对象(instance object)的isa指针指向的类对象(class object)里面还有一个isa呢？_

这个类对象(class objec)的isa指向的依然是一个objc-class，它就是“元类对象”(metaclass object)，它和类对象(class object)的关系是这样的:  类对象(class object)中包含了类的实例变量，实例方法的定义，而元类对象(metaclass object)中包括了类的类方法(也就是C++中的静态方法)的定义。类对象和元类对象中水果公司当然还会包含一些其它的东西，以后也可能添加其它的内容，但对于我们了解其内存布局来说，只需要记住：类对象存的是关于实例对象的信息(变量，实例方法等)，而元类对象(metaclass object)中存储的是关于类的信息(类的版本，名字，类方法等)。要注意的是，类对象(class object)和元类对象(metaclass object)的定义都是objc_class结构，其不同仅仅是在用途上，比如其中的方法列表在类对象(instance object)中保存的是实例方法(instance method)，而在元类对象(metaclass object)中则保存的是类方法(class method)。



### 3. 图例说明

![](http://img.my.csdn.net/uploads/201210/21/1350831500_2327.jpg)
图中可以看出，D3继承D2,D2继承D1,D1最终继承NSObject。下图从D3的一个对象开始，排列出D3 D2 D1 NSObject 类对象，元类对象等关系。

![](http://img.my.csdn.net/uploads/201210/21/1350831599_3230.png)
图中的箭头都是指针的指向。



参考：

[Objective-C内存布局](http://www.cnblogs.com/csutanyu/archive/2011/12/12/Objective-C_memory_layout.html)

[Objective-C Runtime](https://developer.apple.com/reference/objectivec/1657527-objective_c_runtime?language=objc)



