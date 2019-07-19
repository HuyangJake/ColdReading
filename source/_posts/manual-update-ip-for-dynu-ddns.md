---
title: Shell 脚本更新公网 IP 到 dynu DDNS 服务
date: 2019-07-17 10:19:44
tags: [网络]
---

dynu更新ip的服务尽然被Q了, 使用代理进行更新ip，但更新的是代理服务器的ip地址。

基于以上的矛盾点，我想到的解决思路是：

1. 不使用代理获取真实的公网IP并保存到变量
2. 设置代理
3. 将真实IP变量手动更新到dynu服务
4. 定时执行此脚本

<!-- more -->

前3步具体实现如下

``` shell
#!/bin/bash

echo $SHELL

#验证拿到的真实IP是否合法
function isValideIP() {
	regex="\b(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9][0-9]|[1-9])\.(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9][0-9]|[0-9])\.(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9][0-9]|[0-9])\.(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9][0-9]|[1-9])\b"
	ckStep2=`echo $1 | egrep $regex | wc -l`
	if [ $ckStep2 -eq 0 ]
	then
		echo 0
	else
	    echo 1
	fi
}

#用于获取真实IP的服务，由于是些三方的服务，添加多个避免其中某个失效
paths=("whatismyip.akamai.com" "ifconfig.me" "icanhazip.com" "ident.me" "ipecho.net/plain" "myip.dnsomatic.com")
count=${#paths[*]} 
index=0
while [ $index -lt $count ];
do
	unset all_proxy
	ip=`curl --connect-timeout 5 -m 5 ${paths[$index]}`
	echo "using ${paths[$index]}"
	res=$(isValideIP $ip)
	if [ $res -eq 1 ]; then
		echo "ipaddr: $ip"
		time=$(date "+%Y-%m-%d %H:%M:%S")
		echo $time
		export all_proxy=socks5://127.0.0.1:1086
		echo `curl "https://api.dynu.com/nice/update?username=yourusername&password=yourpwd&hostname=your_domain&myip=$ip"`
		let index=count
	else
		echo "fail"
		let index+=1
	fi
done
```

注意执行 shell 脚本用的 zsh 还是 bash，因为要涉及到设置到代理和取消代理。

第4步可以使用 cron 命令或者是 mac 的 launchctl 来创建定时操作任务，不在本文中介绍。

Mac中使用 launchctl 的流程可以查看 [Mac 创建定时任务](http://www.yangjie.hu/2019/07/16/launchctl-in-mac/)