---
title: Apple debugging---使用LLDB替换方法为自定义的block
date: 2017-05-25 22:26:02
tags: 
categories: [猿猿养成记, 学习笔记]
---

### Swizzling with block injection

要在`lldb`中使用Objective-C的runtime，可以通过导入头文件来获得运行时的各种黑色科技。

	po @import Foundation

<!--more-->
	
在控制台输入`po`

	（lldb) po 
	Enter expressions, then terminate with an empty line to evaluate:
		1:
		
然后可输入多行代码，就像在Xcode中写代码一样。在编写完毕后，回车空一行，再回车代码就会被执行了。下面是一个例子：

	1: @import Cocoa; 
  	2: id $class = [NSObject class]; 
  	3: SEL $sel = @selector(init); 
 	4: void *$method = (void *)class_getInstanceMethod($class , $sel); 
  	5: IMP $oldImp = (IMP)method_getImplementation($method); 
	
	id (^$block)(id) = ^id(id object) { 
		if ((BOOL)[object isKindOfClass:[NSView class]]) {
			 fprintf(stderr, "%s\n", (char *)[[[object class] description] UTF8String]); 
		} 
		return object;
	}
	IMP $newImp = (IMP)imp_implementationWithBlock($block);
	method_setImplementation($method, $newImp);
	
	
至此，NSObject的init方法就被替换为了上述代码中的block，`LLDB`中有一个bug，在block中执行IMPS就会崩溃。

内容来自：__《Advanced Apple Debugging & Reverse Engineering》__

