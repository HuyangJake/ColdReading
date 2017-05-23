---
title: Apple Debugging 开始使用LLDB
date: 2017-05-23 23:43:58
tags:
---

### Getting around Rootless

Apple的Rootless机制虽然保护了系统的安全，同时也将我们挡在了门外。

### Disabling Rootless

在电脑的Recovery Mode，打开终端输入 

	csrutil disable; reboot

电脑会重启，此时电脑就禁用了Rootless，可以调试系统级别的应用。

<!--more-->

可以使用命令：

 	lldb -n Finder
 	
 看到类似以下信息就表示成功
 
 	Process XXX stoped
 	.
 	.
 	.
 
### Attaching LLDB to Xcode
 使用`lldb` attach 到最常用的工具Xcode！

	file /Applications/Xcode.app/Contents/MacOS/Xcode

 运行程序：

	process lanuch -e /dev/ttys004 --
	
	
这里的`ttys004`是shell中的某个tab的唯一id


 按`Ctrl + C`可以暂停debugger， 添加一个断点：

		breakpoint set -n "-[NSView hitTest:]"
		

		continue //断点后继续执行
		
		po $rdi // 查看RDI CPU register
		
		//修改一个断点，添加一个触发条件
		breakpoint modify 1 -c "(BOOL)[$rdi isKindOfClass:[NSTextView class]]" 