---
title: 一个集群管理 bash 脚本示例
layout: post
tags: cluster management bash ssh scp
category: scripts
---

编写一个集群管理脚本(**cm.sh**)的目的是:

- 保证集群中每个节点的环境是一致的.
- 减少人工操作, 省时省心.


以重新安装模块为例, 任务: 开发调试后, 重编内核模块, 要在集群中验证功能, 需要替换原来的模块, 安装新的模块.

功能上需要保证:

- 旧模块被卸载, 新的模块成功加入, 而且使用的是新模块.


对 cm.sh 的需求:

- 抑制 ssh 登录的 banner: `ssh -q` ([ref](http://serverfault.com/questions/66986/suppressing-ssh-banner-from-openssh-client))
- 抑制 motd: `touch ~/.hushlogin` ([ref1](http://www.debian-administration.org/articles/546), [ref2](http://serverfault.com/questions/36421/stop-ssh-login-from-printing-motd-from-the-client))
- 方便修改: 定义变量, 使用 loop.
- 支持输入 option: 定义函数, 使用 case.
- 支持并行发送命令: `ssh -f` ----> 实际表明, 使用 -f 的效果不如使用 "**&**"
- 支持等待某些进程执行完成, 好处是显示不会错乱+用户能感知完成: 使用 `wait` 指定进程号.


*1,2 两步的目的是让 ssh 没有多余的输出.*  


cm.sh 执行前的假设:

- cm.sh 放置的节点作为 master 节点, 它生产最原始的数据, 并分发给其他节点.
- 每个节点的目录结构是一致的: 源码路径一致, ko 放置位置一致.
- 已经创建了 ~/.hushlogin 文件.
- 建立了 master 对其他节点的 [password-less login](http://rcsg-gsir.imsb-dsgi.nrc-cnrc.gc.ca/documents/internet/node31.html)


cm.sh

<script src="https://gist.github.com/2479091.js"> </script>

一個更好的腳本 better-cm.sh

<script src="https://gist.github.com/2500303.js"> </script>

