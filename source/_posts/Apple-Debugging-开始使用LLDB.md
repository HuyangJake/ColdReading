---
title: Apple Debugging---开始使用LLDB
date: 2017-05-23 23:43:58
tags: 学习笔记
---

### Getting around Rootless

OS X 10.11 中引入的 Rootless，即使是root用户，也无法对以下路径有写和执行的权限：

	/System
	/bin
	/sbin
	/usr (except /usr/local)
	只有Apple自身签名的软件（含命令行工具）可以。

Apple的Rootless机制虽然保护了系统的安全，防止了大部分恶意软件想通过单纯的引导用户，输入自己的密码点击确定来直接切换到root做一些违法的事情。也可以防止黑客黑了一个用户然后就有root权限可以肆意妄为的情况，不过对于真正的黑客来说

![](https://pic2.zhimg.com/e3a51869c1a65e3cb56378470b5af695_b.jpg)



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

方式一：

	lldb file /Applications/Xcode.app/Contents/MacOS/Xcode
	
方式二：

	lldb -n Xcode  //需要Xcode在运行状态
	
方式三：

	pgrep -x Xcode  //需要Xcode在运行状态
	lldb -p 89921 //89921为Xcode的PID
	
方式四：(attaching 一个将会运行的程序)

	lldb -n Finder -w
	
	pkill Finder

 运行程序：

	process lanuch -e /dev/ttys004 --
	
	
这里的`ttys004`是shell中的某个tab的唯一id


 按`Ctrl + C`可以暂停debugger， 添加一个断点：

		breakpoint set -n "-[NSView hitTest:]"
		

		continue //断点后继续执行
		
		po $rdi // 查看RDI CPU register
		
		//修改一个断点，添加一个触发条件
		breakpoint modify 1 -c "(BOOL)[$rdi isKindOfClass:[NSTextView class]]" 
		
		
内容出处：
__《Advanced Apple Debugging & Reverse Engineering》__