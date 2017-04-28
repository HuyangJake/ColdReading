---
title: 解读Objective-C的NSDictionary
date: 2017-04-20 11:27:20
tags: iOS
---

NSDistionary是非常熟悉常用的一个Foundation中的类，最显著的几个特点是：

* 以键值对存储数据
* 存储的数据具有唯一性
* 数据无序


我们这次不是讲NSDictionary或NSMutableDictionary的方法，这些在[Apple API Reference](https://developer.apple.com/reference/foundation/nsdictionary?language=objc)中查询即可了解啦~
我们来探索下NSDictionary的内部

<!--more-->
### Apple文档的启示

苹果貌似并不推荐我们去继承`NSDictionary`来使用其子类（苹果已经实现的子类：`NSMutableDictionary`除外）

>Before making a custom class of NSDictionary, investigate __NSMapTable__ and the corresponding Core Foundation type, __CFDictionary__. Because NSDictionary and CFDictionary are “toll-free bridged,” you can substitute a CFDictionary object for a NSDictionary object in your code (with appropriate casting). Although they are corresponding types, CFDictionary and NSDictionary do not have identical interfaces or implementations, and you can sometimes do things with CFDictionary that you cannot easily do with NSDictionary.

>If the behavior you want to add supplements that of the existing class, you could write a category on NSDictionary. Keep in mind, however, that this category will be in effect for all instances of NSDictionary that you use, and this might have unintended consequences. Alternatively, you could use composition to achieve the desired behavior.

_大致意思是让我们想要给`NSDictionary`添加新功能就使用分类，不要继承子类。如果打算要继承`NSDictionary`，则需要好好研究下`NSMapTable`和相对应的Core Foundation类型：`CFDictionary`_

`CFDictionary`是Core Foundation中的字典，和`NSDictionary`可以无代价转换，两者可以互相调用对方的方法创建和释放。可以使用`CFDictionary`来实现`NSDictionary`不容易实现的功能。·

另外苹果推荐让我们研究的[`NSMapTable`](https://developer.apple.com/reference/foundation/nsmaptable?language=objc)具体是什么呢?

>The NSMapTable class is a mutable collection modeled after NSDictionary

看到这句话刚开始我有点不解，NSMapTable竟是模仿NSDictionary的一个可变得集合，那跟NSMutableDictionary有什么不同啊？

NSMapTable与NSDictionary的几个不同点是：

* 可以weak的方式持有对象，当持有的对象的引用计数为0的时候，该对象会自动从集合中移除
* key 和 value在输入的时候都可以copy，使用指针对做isEqual或者hashing对比
* 可以包含任何指针，不再仅仅局限于Objective-C的对象

NSMapTable的初始化就可以指定 key 和 value 的持有方式

苹果提供了几个便捷的工厂方法
``` objectivec
//key strong， value strong
+ (NSMapTable<KeyType, ObjectType> *)strongToStrongObjectsMapTable NS_AVAILABLE(10_8, 6_0);
//key weak， value strong
+ (NSMapTable<KeyType, ObjectType> *)weakToStrongObjectsMapTable NS_AVAILABLE(10_8, 6_0); // entries are not necessarily purged right away when the weak key is reclaimed
//key strong， value weak
+ (NSMapTable<KeyType, ObjectType> *)strongToWeakObjectsMapTable NS_AVAILABLE(10_8, 6_0);
//key weak， value weak
+ (NSMapTable<KeyType, ObjectType> *)weakToWeakObjectsMapTable NS_AVAILABLE(10_8, 6_0); // entries are not necessarily purged right away when the weak key or object is reclaimed

```

或者像这样指定：

``` objectivec
//还可以指定NSMapTableCopyIn(添加前进行copy) ， 或者NSMapTableObjectPointerPersonality（使用指针去hashing）
 NSMapTable *mapTable = [NSMapTable mapTableWithKeyOptions:NSMapTableStrongMemory valueOptions:NSMapTableWeakMemory];
 
 //需要一个对象到对象的映射集合，可以使用如下的mapTable
 
 NSMapTable *objectToObjectMapping = [NSMapTable mapTableWithStrongToStrongObjects];
```



同样类推，[NSHashTable](https://developer.apple.com/reference/foundation/nshashtable?language=objc)是对NSSet的模仿和衍生，扩展的功能与NSMapTable相同

### 抛两个问题🙈

##### Q：上千的对象存储到NSDictionary时，出现效率低下的问题是什么原因？

直接先回答问题：因为存储的数量很多后hash表会出现比较多的冲突，解决大量冲突会占用大量CPU资源，造成效率低下。

[hash表](https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8)是什么？跟NSDictionary又有什么关系？

NSDictionary是使用hash表来实现key和value之间的映射和存储的。

hash表的基本思想是：将key作为关键字，放到hash函数ƒ(k)中计算得到结果为hash地址，然后将value值的地址存储到hash表中的ƒ(key)的hash地址中。

当关键字集合很大的时候，不同的关键字可能会通过hash函数映射到hash表的同一地址上。key1 ≠ key2, ƒ(key1) = ƒ(key2),这种情况称为冲突。冲突是不可避免的，只能通过改善哈希函数来减少冲突。（构造哈希函数方法请参考[这里](https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8)或查找其他资料）

然而有冲突必定是要解决的，不然新的值怎么存储到哈希表中呢。
处理冲突主要有（查看原文请移步底部参考链接：哈希表）：

* __开放定址法__
	* 线性探测
	* 平方探测
	* 伪随机探测
* __单独链表法__
 
	将散列到同一个存储位置的所有元素保存在一个链表中。实现时，一种策略是散列表同一位置的所有冲突结果都是用栈存放的，新元素被插入到表的前端还是后端完全取决于怎样方便。
	![](http://jpkc.nwu.edu.cn/sjjg/study_online/book/8/4_2.files/image001.gif)
	
* __再哈希__

	上一次哈希计算发生冲突后，使用另外的哈希函数进行哈希计算知道冲突不再发生

* __建立公共溢出区__

	为冲突的哈希地址单独建立一块区域，所有冲突的哈希地址都存放在此区域中


到这里也就可以推断出，如果在NSDictionary存取时出现效率低下的情况，那么很可能是： 1.频繁的扩容 2.冲突过多

##### Q：NSMapTable是怎样的结构呢？

__先来看看NSMapTable的结构__(代码来自于cocotron)
NSMapTable是一个key－value的容器，下面是NSMapTable的部分代码：

``` objectivec
typedef struct {
   NSMapTable        *table;
   NSInteger                i;
   struct _NSMapNode *j;
} NSMapEnumerator;
```

上述结构体描述了遍历一个NSMapTable时的一个指针对象，其中包含table对象自身的指针，计数值，和节点指针。

``` objectivec
typedef struct {
   NSUInteger (*hash)(NSMapTable *table,const void *);
   BOOL (*isEqual)(NSMapTable *table,const void *,const void *);
   void (*retain)(NSMapTable *table,const void *);
   void (*release)(NSMapTable *table,void *);
   NSString  *(*describe)(NSMapTable *table,const void *);
   const void *notAKeyMarker;
} NSMapTableKeyCallBacks;
```
 

上述结构体中存放的是几个函数指针，用于计算key的hash值，判断key是否相等，retain，release操作。

``` objectivec
typedef struct {
   void       (*retain)(NSMapTable *table,const void *);
   void       (*release)(NSMapTable *table,void *);
   NSString  *(*describe)(NSMapTable *table, const void *);
} NSMapTableValueCallBacks;
```
 

上述存放的三个函数指针，定义在对nsmaptable插入一对key－value时，对value对象的操作。

``` objectivec
@interface NSMapTable : NSObject {
   NSMapTableKeyCallBacks   *keyCallBacks;
   NSMapTableValueCallBacks *valueCallBacks;
   NSUInteger             count;
   NSUInteger             nBuckets;
   struct _NSMapNode  **buckets;
}
```

上面是NSMtabtable真正的描述，可以看出来NSMapTable是一个哈希＋链表的数据结构(也就是使用Hash表中的单链表法解决冲突)，因此在NSMapTable中插入或者删除一对对象时

寻找的时间是O（1）＋O（m），m最坏时可能为n。

 O（1）：为对key进行hash得到bucket的位置
 O（m）：遍历该bucket后面冲突的value，通过链表连接起来。

因此：NSDictionary中的key Value遍历时是无序的，至如按照什么样的顺序，跟hash函数相关。NSMapTable使用NSObject的哈希函数。

``` objectivec
-(NSUInteger)hash {
   return (NSUInteger)self>>4;
}
```
上述是NSObject的哈希值的计算方式，简单通过移位实现。右移4位，左边补0.
因为对象大多存于堆中，地址相差4位应该很正常。


参考：

[NSMapTable: more than an NSDictionary for weak pointers](http://www.isaced.com/post-235.html)

[Apple API Reference - NSMapTable](https://developer.apple.com/reference/foundation/nsmaptable?language=objc)

[Apple API Reference - NSHashTable](https://developer.apple.com/reference/foundation/nshashtable?language=objc)

[维基百科 - 哈希表](https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8)(需要翻下墙，也可以问下度娘)

[关于NSDictionary](http://ibloodline.com/articles/2016/07/11/NSDictionary.html)

[NSDictionary 的内部实现](http://www.cnblogs.com/doudouyoutang/p/4379068.html)

_《Objective-C高级编程：iOS与OS X多线程和内存管理》 ARC的实现_
