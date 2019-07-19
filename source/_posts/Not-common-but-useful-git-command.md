---
title: 不常用但实用的Git操作
date: 2019-04-15 15:55:11
tags:
---

“不常用只是对我个人而言，不喜勿喷~”
那么我先列一下要讲到的git命令吧

``` shell
git rebase
git rebase --onto
git reset --soft
git pick
```

<!-- more -->

### Git rebase

中文叫 “变基”（从sourcetree上的翻译得来），单从名字理解起来也挺形象的，就是将当前的基础给修改掉，移花接木之类的意思吧😂

`rebase` 实现的最终效果是跟 `merge` 相同————将两个分支的修改合并。但略有不同，比如在新的分支中做了 C5 C6 的两次修改提交，现在要将这两个修改合到原分支上。 从下面两张图来看 `rebase` 和 `merge` 的区别吧：

<!-- more -->

__merge__

![](media/15553149347458/15553154431940.jpg)

__rebase__
![](media/15553149347458/15553154578706.jpg)


`rebase`操作之后将原有新分支上的 C5 C6 提交清除（C5 C6并不会马上就被丢弃，如果运行垃圾收集命令`pruning garbage collection`, 这些被丢弃的提交就会删除），最后只剩下了原分支。这样的做法可以简化分支树。


#### 如何使用

知道什么区别之后，我们了解下如何使用：

``` shell
#当前处于 develop 分支
$ git checkout mywork
$ git rebase develop
```

`rebase`的过程和可能会出现冲突，`rebase`会暂停这时候需要解决冲突，完成之后，使用 `git add` 更新这些内容的索引

此时不需要进行`commit` 只需要执行以下命令，`rebase`就会继续完成操作。

``` shell
$ git rebase --continue
```

在任何时候，你可以用--abort参数来终止rebase的行动，并且"mywork" 分支会回到rebase开始前的状态。

``` shell
$ git rebase --abort
```

### Reference

[git rebase 使用详解](https://blog.csdn.net/chenansic/article/details/44122107)
[git 中文讲解](http://gitbook.liuhui998.com/4_2.html)
[Git删除commit提交的log记录](https://www.cnblogs.com/zqunor/p/8620335.html)