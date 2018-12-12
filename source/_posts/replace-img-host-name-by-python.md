---
title: 一键替换博客所有图片域名 
date: 2018-10-27 15:53:01
tags: [随笔]
---

首先介绍下我的博客[冷读空间](http://huyangjie.com)，是一个使用hexo生成的静态博客。

博客中的图床一直使用的是七牛云的对象存储服务，我的使用方式很简单只是图片上传好之后获得一个七牛对象存储空间的默认外链地址，就放到博客中使用。完全没有什么技术含量😂

### 问题的诞生

前段时间被告知七牛要回收默认的测试域名，这就意味着所有历史文章中的图片都要挂了。必须要绑定自己的域名到对象存储空间，并替换所有历史文章中对图片的引用地址。

<!--more-->

问题就这么产生了，博客文章数量那么多（其实我并不多，就是懒），手动地去搜索替换多麻烦。首先想到的救星是`mweb`，这个强大的博客编辑工具，它的外部文档功能如果能进行全局搜索替换这个事情就简单了。

有时候人就是该磨砺一下才不会那么懒惰——搜索和替换只能单文件进行操作。我也没有再去找其他工具去实现这工作。打算自己写个简单脚本

### 解决方案

使用苹果自带的Python2.7 环境成功跑通完成任务。

脚本中做的主要任务有三个：

1. 获取指定目录下的所有文件名
2. 读取博客文章文件内容
3. 替换文章内容中指定字符串

脚本写得很粗糙，还是不要脸得贴上来了。

``` python
# -*- coding: utf-8 -*-
import os

old_str = '*******'

new_str = '*******'

dir_path = 'resource-path'

def read_file(file_path, name):
	count = 0
	file_data = ''
	with open(file_path, 'r') as f:
		for line in f:
			if old_str in line:
				line = line.replace(old_str,new_str)
				count += 1
			file_data += line
	print('--------文章：' + name + '------修改域名数量：' + str(count).encode('utf-8'))
	replace_content(file_path, file_data)

def replace_content(file_path, file_data):
	with open(file_path, 'w') as f:
		f.write(file_data)	
	
def get_all_files(path):
	files = os.listdir(path)
	for file_name in files:
		if file_name[-2:] == 'md':
			read_file(os.path.join(path, file_name), file_name)

if __name__ == '__main__':
	get_all_files(dir_path)
```

成果：

``` shell
➜  blog python replace_hostname.py
--------文章：Apple-Debugging-开始使用LLDB.md------修改域名数量：0
--------文章：Hi！CALayer我们聊聊.md------修改域名数量：2
--------文章：JSPatch遇上Swift.md------修改域名数量：0
--------文章：一次版本发布的启示.md------修改域名数量：3
--------文章：1024.md------修改域名数量：0
--------文章：小程序逻辑层.md------修改域名数量：1
--------文章：机器学习-统计基础-t检验（2）.md------修改域名数量：0
--------文章：机器学习-统计基础-假设检验.md------修改域名数量：0
--------文章：Apple-debugging-使用LLDB替换方法为自定义的block.md------修改域名数量：0
--------文章：Objective-C-isa指针.md------修改域名数量：0
--------文章：自定义NSURLProtocol实现UIWebView缓存机制.md------修改域名数量：0
--------文章：Objective-C中的装饰模式.md------修改域名数量：6
--------文章：机器学习-统计基础-归一化-正态分布-抽样分布.md------修改域名数量：1
.
.
.
```

替换完成后，再发布文章线上的图就都乖乖回来了。

此文章权当为简单的脚本尝试做个记录吧😂

