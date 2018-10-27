---
title: iOS多线程安全---属性篇
date: 2018-08-25 09:33:02
tags: [iOS]
---

APP内涉及到多线程的设计难免会增加Debug的难度。但其实多线程共同访问同一个对象或是一段代码，在编程过程中经常用到，这就需要考虑到多线程安全。

#### Property

``` objectivec
@property (nonatomic, strong) NSString *userName;
```

上面定义属性是我们工作中常用的方式，印象里只是`nonatomic`是非原子性，可以提高性能。而`atomic`是Property的原子性修饰，开发中一般不使用，原因是`atomic`原子性修饰在大多数情况下没有这样的需求，并且频繁调用会影响性能，另外`atomic`更严格地说并不能绝对保证线程的安全。

<!--more-->

---- 
Why？

先了解`atomic`的具体实现：

`atomic`的本质其实是对getter和getter方法加锁，使用的是`spinlock_t`自旋锁。

具体的实现在`runtime`源码中的`objc-accessors.mm`文件

getter方法实现

``` objectivec
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
        
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}
```

setter方法实现

``` objectivec
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}

```


---

#### 值类型Property
先以BOOL值类型为例，当我们有两个线程访问如下property的时候：

``` objectivec
@property (nonatomic, assgin) BOOL    isDeleted;

//thread 1
bool isDeleted = self.isDeleted;

//thread 2
self.isDeleted = false;
```

线程1和线程2，一个读(load)，一个写(store)，对于BOOL isDeleted的访问可能有先后之分，但一定是串行排队的。而且由于BOOL大小只有1个字节，64位系统的地址总线对于读写指令可以支持8个字节的长度，所以对于BOOL的读和写操作我们可以认为是原子的，所以当我们声明BOOL类型的property的时候，从原子性的角度看，使用atomic和nonatomic并没有实际上的区别（当然如果重载了getter方法就另当别论了）。

如果是int类型呢？

``` objectivec
@property (nonatomic, assgin) int    count;

//thread 1
int curCount = self.count;

//thread 2
self.count = 1;
```

同理int类型长度为4字节，读和写都可以通过一个指令完成，所以理论上读和写操作都是原子的。从访问内存的角度看nonatomic和atomic也并没有什么区别。[多线程访问内存](#memory)

__那么设置`atomic`的用处？__

设置atomic之后，默认生成的getter和setter方法执行是原子的。也就是说，当我们在线程1执行getter方法的时候（创建调用栈，返回地址，出栈），线程B如果想执行setter方法，必须先等getter方法完成才能执行。举个例子，在32位系统里，如果通过getter返回64位的double，地址总线宽度为32位，从内存当中读取double的时候无法通过原子操作完成，如果不通过atomic加锁，有可能会在读取的中途在其他线程发生setter操作，从而出现异常值。如果出现这种异常值，就发生了多线程不安全。

然而设置了`atomic`也不一定安全

``` objectivec
@property (atomic, assign)    int       intA;

//thread A
for (int i = 0; i < 10000; i ++) {
    self.intA = self.intA + 1;
    NSLog(@"Thread A: %d\n", self.intA);
}

//thread B
for (int i = 0; i < 10000; i ++) {
    self.intA = self.intA + 1;
    NSLog(@"Thread B: %d\n", self.intA);
}
```

即使将intA声明为atomic，最后的结果也不一定会是20000。原因就是因为`self.intA = self.intA + 1;`不是原子操作，虽然intA的getter和setter是原子操作，但当我们使用intA的时候，整个语句并不是原子的，这行赋值的代码至少包含读取(load)，+1(add)，赋值(store)三步操作，当前线程store的时候可能其他线程已经执行了若干次store了，导致最后的值小于预期值。这种场景我们也可以称之为 __多线程不安全__。

#### 指针Property指向的内存区域
这一类多线程的访问场景是我们很容易出错的地方，即使我们声明property为atomic，依然会出错。因为我们访问的不是property的指针区域，而是property所指向的内存区域。可以看如下代码：

``` objectivec
@property (atomic, strong) NSString *stringA;

//thread A
for (int i = 0; i < 100000; i ++) {
    if (i % 2 == 0) {
        self.stringA = @"a very long string";
    }
    else {
        self.stringA = @"string";
    }
    NSLog(@"Thread A: %@\n", self.stringA);
}

//thread B
for (int i = 0; i < 100000; i ++) {
    if (self.stringA.length >= 10) {
        NSString* subStr = [self.stringA substringWithRange:NSMakeRange(0, 10)];
    }
    NSLog(@"Thread B: %@\n", self.stringA);
}
```

虽然`stringA`是`atomic`的property，而且在取substring的时候做了length判断，线程B还是很容易crash，因为在前一刻读length的时候`self.stringA = @"a very long string";`，下一刻取substring的时候线程A已经将`self.stringA = @"string";`，立即出现out of bounds的Exception，crash，— __多线程不安全__。


----

* 为什么值类型不太需要考虑线程问题
    * 值类型的赋值都是深拷贝，是两个独立的对象。
* 值类型和引用类型并不是绝对独立的
    * 值类型嵌套值类型
    * 值类型嵌套引用类型
    * 引用类型嵌套引用类型
    * 引用类型嵌套值类型


---

#### <span id=memory>多线程内存访问</span>

先来看下多线程是如何同时访问内存的。不考虑CPU cache对变量的缓存，内存访问可以用下图表示：

![](http://www.mrpeak.cn/images/safe02.png)


从上图中可以看出，我们只有一个地址总线，一个内存。即使是在多线程的环境下，也不可能存在两个线程同时访问同一块内存区域的场景，内存的访问一定是通过一个地址总线串行排队访问的，所以在继续后续之前，我们先要明确几个结论：

* 结论一：内存的访问时串行的，并不会导致内存数据的错乱或者应用的crash。

* 结论二：如果读写（load or store）的内存长度小于等于地址总线的长度，那么读写的操作是原子的，一次完成。比如bool，int，long在64位系统下的单次读写都是原子操作。

---
>对于平时编写应用层多线程安全代码，建议还是多使用@synchronized，NSLock，或者dispatch_semaphore_t，多线程安全比多线程性能更重要，应该在前者得到充分保证，犹有余力的时候再去追求后者。

---

参考文章：

* [iOS多线程到底不安全在哪里？](http://mrpeak.cn/blog/ios-thread-safety/)
* [关于synchronized](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)
* [理解iOS中的锁](https://bestswifter.com/ios-lock/)
* [OC中atomic属性如何保证线程安全](https://www.jianshu.com/p/574f2223ccb0)
* [runtime库](https://opensource.apple.com/)