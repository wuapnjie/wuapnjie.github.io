---
title: Daily Linux
date: 2016-09-08 08:56:01
tags: [Linux,Daily]
---

这篇文章会记录一些常用的Linux命令。

``` bash
#ssh连接address的pi用户
ssh [address] -l pi
ssh pi@[address]    

#ssh连接中的文件传输命令
scp [file] user@[address]:/dir

#生成ssh公钥
ssh-keygen

#拷贝公钥至远程服务器
ssh-copy-id user@[address]

#ping所有内网下的IP,可以用来查看树莓派的IP
nmap -sP 192.168.1.1/24

#打开vncserver,:1表示端口微5901，先要安装tightvncserver
vncserver :1

#改变屏幕伽马值
xgamma -gamma .75

#查看Linux操作系统位数
file /bin/bash
#或者
getconf LONG_BIT

#配置java版本
sudo update-alternatives --config java

#打印当前目录
pwd

#设置默认编辑器
export EDITOR=/usr/bin/vim
```
