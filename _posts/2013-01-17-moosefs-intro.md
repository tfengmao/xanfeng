---
title: MooseFS浅析
layout: post
category: coding
tags: moosefs dfs 
---

[MooseFS](http://www.moosefs.org/)是一个“不是很有名气”的文件系统，至少我以前没有听说过它。它不是一个内核态的分布式FS，而是一个用户态的FS。因此可以想见，它不直接接触快设备，它操作其他块设备FS。  

MooseFS是一个POSIX-compliant的文件系统，因此使用起来和ext3等感觉无差别。除此之外，它拥有其他DFS一样的特点：  
1、HA，通过数据备份实现。  
2、存储空间自动**在线**扩展，这一块MooseFS做得不错，新增一个chunkserver之后，mfsmaster上立即可以看到空间的增长。  
3、快照。这块没有了解。  
4、文件系统级别的trashbin。这个必要性为我怀疑。  

可以看到，MooseFS并不是一个很有特点的DFS，尤其它是single master的结构，虽然有metalogger作为备份，但这个设计肯定会为人诟病**单点故障**，而且**应该**的确会有扩展性问题。  

然而，对我来说，MooseFS有一个显著的优点：便利的安装部署以及管理。  
安装部署：  
官方甚至有[中文版Step-by-step指南](http://pro.hit.gemius.pl/hitredir/id=nGDgjQgQG2WJtgQsdMEY8_VzLXUdQae4lPyomd0nuN..j7/url=moosefs.org/tl_files/manpageszip/moosefs-step-by-step-tutorial-cn-v.1.1.pdf)，是我所见**最清晰、最具可操作性**的一份指南！不过有点需要注意：管理界面web server必须使用mfs用户启动，否则访问时会收到"403 Forbidden"反馈。  
管理：  
主要指其web页面，如下图。  
![](http://blog.opennebula.org/wp-content/uploads/2011/04/moosefs-status.png)  

通过这个界面可以清晰地看到：1）有哪些chunkserver，各自的总容量及可用容量；2）客户端列表，包含ip、mount point及访问权限；3）客户端操作统计，比如statfs的个数；4）其他常见的统计，如节点CPU和内存的走势等。  

不过遗憾的是，MooseFS仍然不能跟踪每一个文件。——当然，这个需求是否合适，也是一个问题。  

至于MooseFS的内部实现，目前没有去分析。
