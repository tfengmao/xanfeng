---
title: Linux 串口(serial port)
layout: post
tags: linux kernel serial port capture crash
category: linux
---

串口，又称串行接口或串行通信结构，是指数据一位位地顺序传送，其特点是通信线路简单，但是速度较慢。

接触到串口是因为kernel crash时，putty无法体现server的响应，此时可以通过串口将日志信息发往其他机器。

与记录内核故障信息相关的程序有[minicom](http://www.cyberciti.biz/tips/connect-soekris-single-board-computer-using-minicom.html)，如果你的机器遵循[IPMI](http://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface)(Intelligent Platform Management Interface)协议，则还可使用[ipmitool](http://ipmitool.sourceforge.net/)来做crash信息收集。

IPMI是在服务器之上有一直连的BMC(Baseboard management controller)设备，猜测BMC包含一个极其精简的kernel，可以访问服务器的硬件设备。

我希望全面记录kernel故障信息，使用ipmitool时，执行如下步骤：  
1、重定向串口。不同的OS处理方式是不同的，可搜查官方文档。  
2、在远端机器上执行命令：

    # ipmitool –I lanplus –H <BMC_IP> -U <usrname> –P <passwd> sol activate
    
其中sol是“serial over lan”的缩写。

*希望从本文始，我能“理论+实践”，慢慢的掌握kernel panic分析的方法。*

**参考资料**  
- [Linux 下串口编程入门](http://www.ibm.com/developerworks/cn/linux/l-serials/)  
- [Serial Programming Guide for POSIX Operating Systems](http://www.easysw.com/~mike/serial/serial.html#2_1)  
