---
title: Mac 创建定时任务(launchctl)
date: 2019-07-16 9:22:04
tags: [技术杂谈]
---

### 目录介绍
- ~/Library/LaunchAgents 由用户自己定义的任务项 
- /Library/LaunchAgents 由管理员为用户定义的任务项 
- /Library/LaunchDaemons 由管理员定义的守护进程任务项 
- /System/Library/LaunchAgents 由Mac OS X为用户定义的任务项 
说明:Agents文件夹下的plist是需要用户登录后，才会加载的，而Daemons文件夹下得plist是只要开机，可以不用登录就会被加载

<!-- more -->

### 任务操作

修改plist文件之后需要重新加载plist文件，再进行启动任务。

``` shell
#加载plist任务
launchctl load /Library/LaunchDaemons/ioser.store.update.ip.plist

---
launchctl load -w   **.pist #设置开机启动并立即启动改服务
launchctl load **.pist #设置开机启动但不立即启动服务 

#启动plist任务
launchctl start ioser.store.update.ip

#卸载plist任务
launchctl unload /Library/LaunchDaemons/ioser.store.update.ip.plist


```

### Reference
[美餐提醒](https://www.cnblogs.com/hanlingzhi/p/6505967.html)