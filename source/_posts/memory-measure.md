---
title: iOS/MacOS内存的测量
date: 2018-12-03 11:56:33
tags:
---

iOS设备的内存大小：
![](https://cdn-images-1.medium.com/max/1600/0*1g0cGhawKJnTO6sA.png)

<!-- more -->

### Xcode计量表

![-w1145](http://note.huyangjie.cn/media/15438090904545/15438193271840.jpg)

### 命令行工具

#### top命令

```
Processes: 411 total, 5 running, 406 sleeping, 2126 threads                      15:12:53
Load Avg: 3.24, 3.04, 3.10  CPU usage: 18.22% user, 6.71% sys, 75.5% idle
SharedLibs: 277M resident, 70M data, 137M linkedit.
MemRegions: 83990 total, 7653M resident, 245M private, 2352M shared.
PhysMem: 15G used (2366M wired), 1377M unused.
VM: 2387G vsize, 1297M framework vsize, 0(0) swapins, 0(0) swapouts.
Networks: packets: 249379/143M in, 128089/35M out.
Disks: 553235/5857M read, 1249971/16G written.

PID   COMMAND      %CPU  TIME     #TH    #WQ  #PORT MEM    PURG   CMPR PGRP PPID STATE
637   webstorm     2.0   02:49.89 60     3    354   665M   64K    0B   637  1    sleeping
5033  lldb-rpc-ser 0.0   00:10.29 8      1    69    625M   0B     0B   5033 1228 sleeping
0     kernel_task  6.7   06:49.77 140/8  0    0     600M-  0B     0B   0    0    running
228   WindowServer 10.9  04:13.65 10/1   3    4431+ 464M-  17M-   0B   228  1    running
1228  Xcode        1.6   01:52.85 20     9    890   319M+  9204K- 0B   1228 1    sleeping
1257  XCBBuildServ 0.0   01:17.54 3      3    44    308M   0B     0B   1257 1228 sleeping
574   Paste        94.6  22:28.97 8      4    347+  303M-  103M+  0B   574  1    sleeping
2434  WeChat       2.1   00:48.94 21     8    727   252M-  21M    0B   2434 1    sleepin
```

__排序__
top命令界面下(Linux): 

p 键 - 按 cpu 使用率排序
m 键 - 按内存使用量排序

在Mac下，先输入 o，然后输入 cpu 则按 cpu 使用量排序，输入 rsize 则按内存使用量排序。

查看物理内存在系统级别上实际占用情况： PhysMem

```
➜  ~ top -l 1 | head -n 10 | grep PhysMem
PhysMem: 13G used (2309M wired), 2908M unused.
```

#### vm_stat命令

使用Python脚本进行解析和包装，输出结果：

```
Wired Memory:		2483 MB
Active Memory:		6814 MB
Inactive Memory:	5648 MB
Free Memory:		271 MB
Real Mem Total (ps):	15974.336 MB
```
`wired, active, inactive, free`分别是什么意义：

简单的说，Mac OS X的[内存]使用情况分为：`wired`, `active`, `inactive` 和 `free` 四种。 　　
`wired`是系统核心占用的，永远不会从系统物理[内存]种驱除。
`active`表示这些[内存]数据正在使用中，或者刚被使用过。
`inactive`表示这些[内存]中的数据是有效的，但是最近没有被使用。
`free`, 表示这些[内存]中的数据是无效的，这些空间可以随时被程序使用。

>当free的[内存]低于某个值（这个值是由你的物理[内存]大小决定的），系统则会按照以下顺序使用inactive的资源。首先 如果inactive的数据最近被调用了，系统会把它们的状态改变成active,并接在原有active[内存]逻辑地址的后面, 如果inactive的 [内存]数据最近没有被使用过，但是曾经被更改过而还没有在硬盘的相应虚拟[内存]中做修改，系统会对相应硬盘的虚拟[内存]做修改，并把这部分物理[内存]释放为free供程序使用。如果inactive[内存]中得数据被在映射到硬盘后再没有被更改过， 则直接释放成free。最后如果active的[内存]一段时间没有被使用，会被暂时改变状态为inactive。 　　 


所以说，如果你的系统里有少量的free memeory和大量的inactive的memeory，说明你的[内存]是够用的，系统运行在最佳状态，只要需要,系统就会使用它们，不用担心。

而反之如果系统的free memory和inactive memory都很少，而active memory 很多，说明你的[内存]不够了。

当然一开机，大部分[内存]都是free,这时系统反而不在最佳状态，因为很多数据都需要从硬盘调用，速度反而慢了。

### 活动监视器





### Reference 

[Mac下的top](https://blog.devtang.com/2011/12/27/mac-top/)
[MAC上命令行查看系统内存使用量](http://smilejay.com/2014/06/mac-memory-usage-command-line/)
