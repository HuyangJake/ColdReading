---
title: Git 仓库文件过大无法上传
date: 2019-04-15 19:33:58
tags: [技术杂谈]
---

Git仓库默认是有100M文件大小的限制，比如我这个项目中的`Python.framework`文件大小超过了100M,就不能上传到GitHub了（这个限制其实可以取消）。

``` shell
remote: error: GH001: Large files detected. You may want to try Git Large File Storage - https://git-lfs.github.com.
remote: error: Trace: 44bd870a4d27f7109aee59495836e701
remote: error: See http://git.io/iEPt8g for more information.
remote: error: File ThirdLibs/Python/Python.framework/Versions/3.4/Python is 132.98 MB; this exceeds GitHub's file size limit of 100.00 MB
```

但更严重的事是 设计图，视频等大文件在项目中存在多了，项目会变得十分臃肿，对日常开发的分支切换，文件替换造成了很多负担。如何解决？见上方代码块中Git的错误提示，给了解决方案: [LFS](https://github.com/git-lfs/git-lfs)

<!-- more -->

解决方案在 [https://git-lfs.github.com](https://git-lfs.github.com) 使用 `Large File Storage`

>Git LFS（Large File Storage, 大文件存储）是可以把音乐、图片、视频等指定的任意文件存在 Git 仓库之外，而在 Git 仓库中用一个占用空间 1KB 不到的文本指针来代替的小工具。通过把大文件存储在 Git 仓库之外，可以减小 Git 仓库本身的体积，使克隆 Git 仓库的速度加快，也使得 Git 不会因为仓库中充满大文件而损失性能。
>

### 使用方法

使用 git lfs track 追踪需要使用 Git LFS 管理的文件。如：

``` shell
git lfs track "*.psd"
```

也可以手动编辑 Git 仓库根目录下的 .gitattributes 文件，例如：

``` shell
*.psd filter=lfs diff=lfs merge=lfs -text
```

修改完成 `.gitattributes` 文件之后将其添加到Git管理中。

__接下来的步骤 我踩了很久的坑！！☹️__

看了很多的使用教程都是说此时再将已经添加到Git中，进行Push操作就可以完成LFS的工作，大文件就可以上传了。

而我使用的实际情况却不是这样，怎么上传都还是出现了与文章开头一样的提示。

经过我的尝试，发现必须要遵循以下的操作流程才可以正常地使用LFS：

> 准备状态: __大文件尚未添加到仓库中__
> 1. 添加 `.gitattributes` 文件编写内容
> 2. 添加 `.gitattributes` 文件到Git仓库管理
> 3. 推送本地仓库到远程地址
> 4. 添加大文件到仓库中
> 5. commit添加操作， Push到远程仓库


##### PS : 关于clone带LFS文件的仓库
目前最新版本的 `git clone` 已经能够提供与 `git lfs clone` 一致的性能，因此自 `Git LFS 2.3.0` 版本起，`git lfs clone` 已不再推荐使用。

在我真实使用的时候，我需要在仓库中初始化 `lfs` 之后再 使用 `git lfs clone` 或者是 `git lfs pull` 才能正确拉下代码。


#### 一些进阶的使用
例如： 
* 只获取仓库本身，不获取LFS对象
* 仅获取指定目录下的LFS对象
...

点击 Reference 中的连接查看

---

### Reference

[Git LFS 操作指南](https://zzz.buzz/zh/2016/04/19/the-guide-to-git-lfs/)