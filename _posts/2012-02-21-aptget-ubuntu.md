---
title: apt-get @ubuntu
layout: post
tags: ubuntu apt-get
category: desktop
---

**PPA**
apt-get 更新软件的时候,  有时会出现如下内容:  
	xan@xmachine:~$ sudo apt-get update
	sudo password for xan: 
	Ign http://ppa.launchpad.net oneiric InRelease 
	Hit http://ppa.launchpad.net oneiric Release.gpg 
	Ign http://mirrors.163.com natty InRelease
这里的 ppa 是什么意思? 摘引 ubuntu 文档 "[什么是 ppa](http://people.ubuntu.com/~happyaron/udc-cn/lucid-html/ch11s02.html)":  
> PPA，表示 Personal Package Archives，也就是个人软件包集.  
有很多软件因为种种原因，不能进入官方的 Ubuntu 软件仓库。 为了方便 Ubuntu 用户使用，launchpad.net 提供了 ppa，允许用户建立自己的软件仓库， 自由的上传软件。PPA 也被用来对一些打算进入 Ubuntu 官方仓库的软件，或者某些软件的新版本进行测试.  
PPA 上的软件极其丰富，如果 Ubuntu 官方仓库中缺少您需要的某款软件，可以去 PPA 上找找看.


