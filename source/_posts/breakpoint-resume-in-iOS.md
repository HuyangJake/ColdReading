---
title: AFNetworking 实现断点续传
date: 2018-05-15 23:14:19
tags: [iOS]
---



### 思路

>1. 从协议层面来说，断点续传的实现都是通过http头里面的Range来实现的。对于以前asi的实现和基于NSURLConnection的实现来说，我们都是通过手动保存当前大小然后填充到Range的策略来实现断点续传功能的。

>2. NSURLSession内置了对于断点续传的api支持，主要是通过resumeData和tmp中间文件。在NSURLSession暂停时，其中间文件的相关信息被保存到resumeData，我们只要通过这个resumeData，即可从之前的下载进度中恢复。也就是说1中的细节被封装在api内部了。

>目前来看，基于NSURLSession的断点续传主要有两个问题需要考虑：

>a. 程序退出（手动kill等）时需要自动保存resumeData，后续应用起来时再恢复之。

>b. iOS10里面的resumeData的保存有点问题，需要特别处理，可以参考此demo:https://github.com/HustHank/BackgroundDownloadDemo.

<!--more-->

上述引用中的

第2点就是我们方案中的App生命周期内的断点续传，只是我们调用的是AFNetworking的API

第1点是我们方案中离线点断续传的思路，不同的是我们使用的是基于`NSURLSession`的`AFNetworking`来实现这个逻辑（`AFNetworking`原生也不支持离线断点续传）。实现的方案跟引用中提到自动保存resumeData也不一致，我们使用文件流保存数据，没有取系统提供的resumeData使用。

----
### 设计流程
![](http://o8ajh91ch.bkt.clouddn.com/QLNetworking_Download.png)

[查看大图](http://o8ajh91ch.bkt.clouddn.com/QLNetworking_Download.pdf)


----
### 文件保存逻辑

断点续传有一个必要的逻辑：

__在开始发起请求任务之前，在本地查找匹配的文件计算下载数据开始的位置。__

这里面有一个隐藏的问题：

>当某个文件在上一次下载任务中已经下载完成，这次发起请求的时候会读取这个文件的大小设置到请求头的Range，从而发起一个不需要下载任何数据的请求。

解决这个问题途径有两个：

* 在触发下载任务之前进行文件分析判断是否需要下载
* 区分下载中和下载完成的路径

1. 前者在实现过程中遇到问题，在请求发起之前没办法切确判断本地读取到的文件数据是否完整。或许可以使用某些类型文件的特性来进行判断。

2. 后者的方式就比较通用简单，简单描述就是将下载未完成的文件放在特定的临时目录，下载完成之后再统一迁移到目标目录，临时目录中的文件也会随之被删除。

我们选择第二种方案，针对iOS的沙盒，temp最适合作为方案中的临时目录。（使用过程中注意`NSFileManager`的线程安全问题）如此能保证断点续传过程中值操作临时文件，不会影响外部流程中对是否需要发起请求的判断。

最后我们的逻辑是：

![](http://o8ajh91ch.bkt.clouddn.com/QLNetworking_download_save.png)

----
### 待优化点
* 后台下载
* 下载文件的校验

### Reference

https://my.oschina.net/snOS/blog/795412

https://blog.csdn.net/u012198553/article/details/53613253

https://github.com/AFNetworking/AFNetworking

https://www.jianshu.com/p/01390c7a4957


