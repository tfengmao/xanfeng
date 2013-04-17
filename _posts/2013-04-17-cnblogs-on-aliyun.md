---
title: 博客园使用阿里云的经验
layout: post
category: cloud
tags: cnblogs cloud aliyun
---

在开始之前，我必须发一下牢骚：次奥，我特么又感冒了！我可是曾经被称为有小强一般的生命力啊！感冒之中，体能感觉还行，但脑力劳动就完全不给力了！效率极差，浪费时间，十分不爽。  
因此告诫自己：慢慢不年轻了，至少不要和疾病正面对抗（打球汗湿了衣服直接穿着捂干、冷天短袖赤脚...)，养成良好的作息习惯（00:00am睡觉deadline、9:00am起床deadline）。  

进入正文。  
本文归纳一下博客园(cnblogs)通过官方博客宣称的使用aliyun云服务器的经验。cnblogs使用aliyun，不管成功失败，我想这都会是一个典型案例。另在[Startup News](http://news.dbanotes.net/item?id=3889)上，这个案例被戏称为“最爱阿里云的还是博客园”，我觉得十分贴切。  
这一系列文章被cnblogs官方称之为“云计算之路”，参见官博标签“[阿里云](http://www.cnblogs.com/cmt/tag/%E9%98%BF%E9%87%8C%E4%BA%91/)”和“[云计算](http://www.cnblogs.com/cmt/tag/%E4%BA%91%E8%AE%A1%E7%AE%97/)”。  
一起还在发展当中...  

###cnblogs使用阿里云的历程

1、2013-03-09开始迁移，主要时间用在了数据库日志的恢复上，正式迁移只用了40分钟，但为了这40分钟准备了1个多月的时间（[ref](http://www.cnblogs.com/cmt/archive/2013/03/09/go-into-cloud.html)）。  
2、3.9，发现访问IP记录都是阿里云的内网IP。  
3、3.12（周二），经受大规模访问考量的第一天，images服务响应缓慢。  
4、3.14，502 Bad Gateway故障。  
5、3.18，阿里云备案系统引起某些子站访问故障。  
6、3.19，Tengine压缩问题。  
7、3.20，SLB引起500。  
8、3.22，阿里云网络存储故障造成网站不能访问。  
9、4.2，CPU和IO“过山车”现象，且监控不到。  
10、4.7，不能发表博文，因为日志高IO引起。  
11、4.13，换RDS。  
12、4.14，触发ADO.NET和SQL Server镜像的BUG。  
  
###cnblogs使用云之前的痛点或使用云的动机     

1、更专心于开发更好的产品、为大家提供更好的服务（[ref](http://www.cnblogs.com/cmt/archive/2013/04/07/3006008.html)）。“只有经历了攒服务器的痛苦，处理服务器故障的煎熬，才能理解我们内心是多么希望云计算的发展”（参见[ref](http://www.cnblogs.com/cmt/archive/2012/12/20/aliyun-server-room.html)的评论区）。  
2、原来的5台服务器分为十几台云服务器，“分”的好处是显而易见的，可以减少互相之间的影响。（[ref](http://www.cnblogs.com/cmt/archive/2013/02/20/aliyun-bandwidth-share.html)）
 
*以下内容来自“[云计算之路：为什么要选择云计算](http://www.cnblogs.com/cmt/archive/2013/02/27/why-into-cloud.html)”。*  

3、BGP线路解决南北互通问题：“我们考虑阿里云，还有一个很大的吸引点——BGP网络线路，联通（之前是网通）与电信之间的线路问题已经困扰我们很长时间。”（参见[ref](http://www.cnblogs.com/cmt/archive/2012/12/20/aliyun-server-room.html)的评论区）。  
4、有效解决硬件单点故障问题。  
5、按需增/减硬件资源。  
6、按需增/减带宽。  
7、更有吸引力的费用支付方式，可以一月一付，节省流动资金。  

###cnblogs的决定：RDS vs. 云服务器

结论：[云计算之路：数据库服务器的选择——舍RDS取云服务器](http://www.cnblogs.com/cmt/archive/2013/02/21/alyun-rds.html)。  

RDS（关系型数据库服务）：  
1、跑在强劲的物理服务器上，硬盘读写速度快：简单来说是跑在物理服务器上的数据库实例。比如针对SQL Server的RDS，阿里云会在服务器上安装好SQL Server，然后把其中的一个SQL Server数据库实例出租给你，并限定该实例所能使用的硬件资源。  
2、维护成本低：由阿里云的DBA负责维护。  
3、成本高，限制多：无法远程控制数据库服务器、一个数据库库实例支持的数据库数量有限制、SQL Server只有2008版本、数据库最大连接数有限制、没有提供大于10G的数据文件的迁移方案（实际>70G）。  

云服务器：  
1、硬盘写入速度是硬伤：阿里云云服务器在磁盘写入时会同时写三份，对写入速度影响比较大。写速度一般在15MB/s，读速度一般在70MB/s。如果对磁盘写入要求高，可以使用RDS。  
2、成本相对低。  

###迁入过程的难题

1、不是复杂，而是繁琐。  
2、阿里云不支持带宽共享。在IDC机房，只需根据实际需求购买一条独享带宽的线路，比如有10台服务器，每台峰值带宽10M，我们不需要购买100M带宽，可能购买80M就够了。使用阿里云，就必须购买带宽峰值和。成本是问题之一，更大的问题是没有弹性分配。([云计算之路：遇到障碍——阿里云不支持带宽共享](http://www.cnblogs.com/cmt/archive/2013/02/20/aliyun-bandwidth-share.html))  
3、是否区分大小写，不同云存储服务商使用的方案不同（根源是Linux和Windows处理策略的不同）：[云计算之路：云存储的纠结](http://www.cnblogs.com/cmt/archive/2013/02/26/upyun-aliyun-url-case-sensitive.html)。  

###cnblogs使用过程中遇到的故障

如非特别指出，下面的“感悟”都是cnblogs的感悟。  

云服务器硬盘IO引起的故障：  
1、  
摘要：4.8故障，以为仍然是其他用户的高磁盘IO影响了cnblogs应用，结果是因为所有日志文件在同一个分区，超过了云服务器磁盘IO的某个极限，造成磁盘IO性能骤降。  
解决办法：将日志文件放到独立的分区上。  
链接：[云计算之路-阿里云上：2013年4月7日14:15~18:35服务器故障经过](http://www.cnblogs.com/cmt/archive/2013/04/08/3006448.html)  
感悟：客服响应仍不够及时，线上问题分秒必争的。“万万没想到，云服务器的硬伤不是在磁盘IO性能低，而是在磁盘IO不稳定。”    
2、  
摘要：3.14 “502 Bad Gateway”。  
解决办法：将问题云服务器迁移到另一个集群。  
链接：[云计算之路-迁入阿里云后：20130314云服务器故障经过](http://www.cnblogs.com/cmt/archive/2013/03/14/2960583.html)  


云服务器负载引起的问题（不确定是因磁盘IO引起的）：  
1、  
摘要：3.12 images.cnblogs.com访问缓慢，很可能因为IIS连接数超过2000引起云服务的强烈反应。  
解决办法：使用多台云服务器，然后对请求做LB，由于没有第二个负载均衡器，使用“DNS轮询+两台云服务器”的方法。  
链接：[云计算之路-入阿里云后：解决images.cnblogs.com响应速度慢的诡异问题](http://www.cnblogs.com/cmt/archive/2013/03/12/2955405.html)  

SLB引起的问题：  
1、  
摘要：某位负载均衡用户的节点被攻击，使得cnblogs的负载均衡受到牵连。  
链接：[云计算之路-阿里云上：终于找出了500错误的真正原因](http://www.cnblogs.com/cmt/archive/2013/03/20/2971868.html)  
感悟：仍然是资源隔离的问题。  
2、  
摘要：IP记录错误，记录的都是阿里云的内网IP。  
链接：[迁入阿里云后遇到的Request.UserHostAddress记录IP地址问题](http://www.cnblogs.com/cmt/archive/2013/03/09/request_userhostaddress-x_forwarded_For.html)  

监控系统的问题：  
1、  
摘要：阿里云监控系统采样间隔长（对外5分钟一次，内部1分钟一次），监测不到CPU和磁盘IO出现的瞬间波动。  
链接：[云计算之路-阿里云上：为什么看不见CPU在坐过山车，磁盘IO在蹦极](http://www.cnblogs.com/cmt/archive/2013/04/06/2997779.html)  
我的感悟：这个应该是不能解决的问题之一吧，只能权衡。  

RDS的问题：  
1、  
摘要：阿里云RDS使用了SQL Server数据库镜像，cnblogs使用了.NET Framework 4.0中的ADO.NET，从而触发了ADO.NET对SQL Server数据库镜像处理上的bug。  
链接：[网站故障公告1：使用阿里云RDS之后一个让人欲哭无泪的下午](http://www.cnblogs.com/cmt/archive/2013/04/16/3024439.html)，[网站故障公告2：找到问题的重要线索](http://www.cnblogs.com/cmt/archive/2013/04/16/3025231.html)，[网站故障公告3：应该找到了问题的真正原因](http://www.cnblogs.com/cmt/archive/2013/04/17/3025409.html)。  
感悟：被微软坑了。  

其他问题：  
1、备案系统问题引起的故障：[阿里云备案系统问题造成部分站点无法正常访问](http://www.cnblogs.com/cmt/archive/2013/03/18/2966317.html)。  
2、Tengine引起的压缩问题：[云计算之路-阿里云上：实战Advanced Logging for IIS分析http内容压缩问题](http://www.cnblogs.com/cmt/archive/2013/03/19/2968947.html)  
3、阿里云存储网络故障造成网站不能访问：[云计算之路-阿里云上：0:25~0:40网络存储故障造成网站不能正常访问](http://www.cnblogs.com/cmt/archive/2013/03/22/2974781.html)  

###cnblogs总结的经验教训

读完这一系列，我觉得cnblogs团队挺厉害，技术过硬、思路清晰、表达明了。比如[解释为了避开云磁盘IO能力差，转而使用RDS](http://www.cnblogs.com/cmt/archive/2013/03/17/aliyun-rds.html)：  

> RDS（Relational Database Service）是阿里云提供的关系型数据库服务，是将直接运行于物理服务器上的数据库实例租给用户，通过对硬件资源的独占分配（这是我们的猜想）避开了云服务器硬盘IO共享带来的性能问题。付出的代价是抛弃了云平台中的关键角色——虚拟化平台。  
> 如果把物理服务器比作发电厂，虚拟化平台就是电网，RDS的解决方案就如同——电网的问题造成供电电压不稳定，于是发电厂直接拉根输电线到用户的家里，不走电网；  
> 如果把物理服务器比作自来水处理厂，虚拟化平台就是公共供水管线，RDS的解决方案就如同——由于某些低楼层用户用水量大，供水水压不够，造成高楼层用户用水困难，于是自来水处理厂直接铺设一根根供水管道到用户家里，不走公共供水管线。

类似的例子在[这个系列](http://www.cnblogs.com/cmt/tag/%E9%98%BF%E9%87%8C%E4%BA%91/)中还有很多，我是越看越佩服cnblogs团队。  

1、用更多的内存弥补磁盘IO性能的不足证明是有效的（[ref](http://www.cnblogs.com/cmt/archive/2013/03/11/2953463.html)）。  
2、负载均衡带来的带宽使用量监控不易（[ref](http://www.cnblogs.com/cmt/archive/2013/03/11/2953463.html)）。  
3、应用的磁盘分配策略，将不同应用部署到不同的磁盘上，提升整体IO性能（[ref](http://www.cnblogs.com/cmt/archive/2013/03/11/2953463.html)）。  
4、多个负载均衡器的需求（[ref](http://www.cnblogs.com/cmt/archive/2013/03/12/2955405.html)）。  
5、资源隔离的需求（[ref](http://www.cnblogs.com/cmt/archive/2013/03/15/2961145.html)）。  
6、**磁盘IO能力和稳定性**。  
7、负载均衡是否优秀（如问题云服务器迁到另一个集群就没问题了）。  
8、要被搜索引擎收录。  
9、出问题后的解决速度，比如虚机迁移的速度。  

悲了个剧的，在写此文的这个下午，我也遇到阿里云lb返回的502错误：  
![](/images/cnblogs-aliyun.jpg)  
以及不止一次遇到503错误："Service Unavailable...HTTP Error 503. The service is unavailable."

迁移速度，阿里云5~15min