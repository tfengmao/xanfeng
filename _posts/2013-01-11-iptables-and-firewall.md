---
title: iptables和防火墙(firewall)
layout: post
category: linux
tags: iptables firewall netfilter
---

防火墙有硬件防火墙和软件防火墙，本文关注的是软件防护墙。Linux下面软件防火墙自2.4内核始，多是基于iptables这一用户态工具。  
网络上有不少iptables相关的资料，中英文都有，但看起来都不是很直观。其实阅读`man iptables`+简单操作，就可以总结出iptables的不同层次：table、chain、rule。  
稍加`man iptables`之后，再阅读鸟哥的“[第九章、防火牆與 NAT 伺服器](http://linux.vbird.org/linux_server/0250simple_firewall.php)”，在鸟哥风趣深入的讲解下，便可达到不错的理解。  

###iptables

这里多引述鸟哥的讲解，谢谢鸟哥！  

**iptables的表和链**:  
1、默认3个表：filter，nat和mangle。  
2、只有默认表才能设置policy。  
3、可以增加新表。  
![](/images/iptables_table_chain.png)  

**包过滤规则是有顺序的**，符合前面的规则后，执行预定的动作，然后不会按序执行后面的规则。因此这个顺序很重要：  
![](/images/iptables_rule_order.png)

各个表之间是有关联的（这部分更多内容参见鸟哥原文了）：  
![](/images/iptables_table_relation.gif)

**iptables的常见操作**：  
从`iptables --help`和manual就可以看到更多信息了，也可以看[鸟哥的原文](http://linux.vbird.org/linux_server/0250simple_firewall.php#netfilter_syntax)，因为解释得很清楚，而且很全面。  

###firewall

这里指代软件防火墙，比如opensuse的[SuSEfirewall2](http://en.opensuse.org/SuSEfirewall2)，它基本上是一个操作iptables的脚本而已。  
