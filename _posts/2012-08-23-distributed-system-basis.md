---
title: 了解分布式系统
layout: post
category: coding
tags: distributed system cloud-computing
---

*OT-1：好久没有写博文，因为：(1)现在下笔要比以前<del>谨慎</del>(本文又冲动了)。(2)更多考虑该去做什么，而不是拿起一个东西就学。另一方面，自己要看的东西大多都看过，已经有初步的理解，剩下的便是实战历练。(3)在针对性地复习算法，从常用到不常用。这是爱好，但不是短期内大成的，重体悟而非记录，所以也没写下来。*

*OT-2：也在纠结工作的事情，不知道未来做什么，没有考虑清楚。*

以下正文。

云计算、大数据...一年一热词，所谓新瓶装旧酒，核心还是分布式技术而已。  
说起分布式，难逃网络通信。这就是一个很复杂的问题，就我的经历来说，没有好的协议和设计，写出来的代码所爆发的问题是爆炸式的。  
相比其他，分布式中的网络问题更为重要。这么多年来，想必已有经验教训、模型可供参考。该了解这些知识，不能自己瞎撞。

有一个"[A Distributed Systems Reading List](http://dancres.org/reading_list.html)"，列出了很多分布式方面的论文书籍。本文是阅读记录。

结构上遵循原文。

###Thought Provokers

1、[Harvest, Yield and Scalable Tolerant Systems](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.33.411) - Real world applications of CAP from Brewer et al

这是99年，斯坦福和伯克利的人发的文章，没什么猛料。它提出两个方向：  
1）妥协(trade-off)。没细看，难逃“为达到A效果，牺牲一定的B效果”。  
2）细分系统，错误隔离。这一条实际上不堪大用，虽然我很认同。它的思路是：把大系统合理地细分，从而把各种错误限制在小模块中，从而提升整体的容错度。这个思路听起来很美，做出来的系统很“干净”，却很需要经验吧。

这是我看的第一篇，它“提出”貌似江湖闻名的“CAP principle”，C-consistency（一致性）、A-high availability（高可用）、P-partition-resilience（容错？集群挂掉一两个节点，不碍事）。  

Strong CAP principle：三者取其二，不能俱全。  
这是例证的，所以挺“神棍的”。  
1）CA without P：有分布式语义的数据库。  
2）CP without A：  
3）AP without C：HTTP web caching。

另外有一个“weak CAP principle”，没有继续了解。

2、[On Designing and Deploying Internet Scale Services](http://www.mvdirona.com/jrh/talksAndPapers/JamesRH_Lisa.pdf) - [James Hamilton](http://www.mvdirona.com/jrh/work/)

James Hamilton，看起来眼熟啊(_ _!!!)，Amazon Web Services Team的，这篇感觉是大招。看后发现，整篇是经验总结，由MSN、Windows Live的分布式网络服务开发中总结而来。比如江湖闻名的“Design for failure”。

也就不逐条列出了，直接看paper吧，[有人](http://blog.csdn.net/blade2001/article/details/6067652)也整理了一个列表。总的来说，没有“革命性”的猛料...  
列出一些重要的：  
1）design for failure  
2）redundancy and fault recovery  
3）commodity hardware slice，这条大概指“一群破机器，能打败大型机”，也就是hadoop那种模式。  
4）single-version software，集群中跑同样的代码，不要去傻呵呵地支持多版本。  
5）Zero trust of underlying components，不要相信底层，不要完全依赖底层来保证稳定。  
6）Do not build the same functionality in multiple components、Keep things simple and robust、keep deployment simple，这几条是类似的，保证简单可依赖（所以，百度今年的口号很靠谱！），这样看起来容易，改起来也容易。同时这些也是“ship offen”的保障，有了这些，才有号称的“敏捷开发”，想很久、设计很久、码很久、调试很久=必死。  
7）Understand the network design、Analyze throughput and latency，理解网络，胸中要有各个部分的平均性能数据。这个网上有个图，把一次读cache、发一条网络消息的大概时间列出来了，就是这个道理。  

3、[Latency Exists, Cope!](http://www.addsimplicity.com/adding_simplicity_an_engi/2007/02/latency_exists_.html) - Commentary on coping with latency and it's architectural impacts

这不是一篇论文，这是一篇博文，它很给力！

关于延时，我是有体会的，在我看来，网络通信、进程调度的不确定性，带来服务响应时间的不可控，尤其网络高压、或者系统问题导致进程hang住时。

这就纠结了，如何处理这种情况才合适，应该还是要“特定情况特定分析”。上面这篇文章给出了一些指导，很受教：  
1）解耦！还是这个，高耦合就挂了！  
2）异步通信，并且考虑带expectation。这条给力！经历过的人都懂的，虽然一般的分布式服务都会考虑到，但就看考虑得好不好，实现得好不好了。  
3）不要依赖一个数据中心。这点就暂不考虑了。  

4、[Latency - the new web performance bottleneck](http://www.igvita.com/2012/07/19/latency-the-new-web-performance-bottleneck/) - not at all new (see Patterson), but noteworthy

这是一篇博文，重点在web时延，不是我关心的，略过。不过这篇博文的呈现方式很有特色。

5、[The Perils of Good Abstractions](http://www.addsimplicity.com/adding_simplicity_an_engi/2006/12/the_perils_of_g.html) - Building the perfect API/interface is difficult

这篇博文的作者和(3)的作者是同一人，内容核心还是那两个字：解耦。

6、[Chaotic Perspectives](http://www.addsimplicity.com/adding_simplicity_an_engi/2007/05/chaotic_perspec.html) - Large scale systems are everything developers dislike - unpredictable, unordered and parallel

作者还是写(3、5)的大叔，文章大致看了看，只是讨论，没有结论性的东西，略过。

7、[Website Architecture](http://poorbuthappy.com/ease/archives/2007/04/29/3616/the-top-10-presentation-on-scaling-websites-twitter-flickr-bloglines-vox-and-more) - A collection of scalable architecture papers from various of the large websites

这是一个合集，包括讲述twitter、Flickr、LiveJournal扩展的slides，搞网站的童鞋不容错过，我就先路过了。

8、[Data on the Outside versus Data on the Inside](http://www.cidrdb.org/cidr2005/papers/P12.pdf) - Pat Helland

微软发的SOA方面的论文。

9、[Memories, Guesses and Apologies](http://blogs.msdn.com/b/pathelland/archive/2007/05/15/memories-guesses-and-apologies.aspx) - Pat Helland

这是一篇博文，题目中“Guesses、Apologies”的含义没去细究。作者的思路，还是那句话，避免“大而全”的东西。看作者的总结：

> Consider the cost/benefit of building big-ass and expensive machines!  Careful design of a collection of replicas may fill the business need at a better value!

10、[SOA and Newton's Universe](http://blogs.msdn.com/pathelland/archive/2007/05/20/soa-and-newton-s-universe.aspx) - Pat Helland

这篇应该是(10)的延伸。

11、[Building on Quicksand](http://arxiv.org/abs/0909.1788) - Pat Helland

这是09年的论文。我们不得不“在浮沙筑高台”。此时怎么办？异步机制。异步带来时延不可控，怎么办？...凉拌...

到目前为止，对于分布式中异步方式带来的时延和错误，我认为没有根本的解决方法，需要根据项目特点细致地对待。

12、[Why Distributed Computing?](http://www.artima.com/weblogs/viewpost.jsp?thread=4247) - [Jim Waldo](http://www.eecs.harvard.edu/~waldo/)

这是03年的博文。从简历看，Jim Waldo是一个达人，经历的知名公司有HP、Sun、VMware，现在在哈佛任教。从他应可深挖资料。

这个短文辩论了即使“有更快的CPU，更快的网络，更大的存储容量，所有请求都可以由一台机器搞定”，分布式计算还是必须的，因为“机器的进化速度<计算要求的增长”。作者调侃：如果有这么一台超强的机器，不需要搞分布式，那么程序员就轻松了，不用再考虑“partial failure”了。

**13、[A Note on Distributed Computing](http://labs.oracle.com/techrep/1994/abstract-29.html) - Waldo, Wollrath et al**

94年的一篇老论文，来自Jim Waldo和他在Sun的同仁们，他们当时好像在做NFS，NFS我不熟悉，但却在快20年后的今天，我仍看到有人在使用它。Sun还真是一个nb的公司，技艺精湛，可惜技术非钱。

对我，目前为止，这篇论文最给力！它对分布式系统的核心问题做了定义，比我已了解的要多、要深入，虽然它只是指出了问题，没有给出解决方法。  
完美的解决方法，可能根本不存在。

看起来，他们一开始的思路是：提供API，隔离本地和分布式操作的区别。这样的API是诱人的，程序员会很喜欢。但实际经验告诉他们，这种抽象必然导致失败，你必须正视本地程序和分布式程序的区别，既不能使用纯分布式的API去抽象本地操作，也不能使用本地形式的API去抽象分布式。

所以Unified Objects是行不通的：  
“The Vision of Unified Objects”  
在分布式环境下，使用object-oriented思路，抽象和隔离底层分布式细节。这样做貌似可以带来好处：1）归约到程序员熟悉的object-oriented设计。2）failure、performance问题限制在模块实现中，不在初始设计中。3）完美的抽象，便利的扩展，适合不同的分布式环境。  
然而，这是自欺欺人的，通过摆弄新概念去规避核心问题（分布式的时延、容错、服务质量保障），必然是不靠谱的。

下面列出论文对分布式计算核心问题的阐述。

分布式计算的核心问题有四：  
1）latency  
2）memory access  
3）partial failure  
4）concurrency  

> partial failure is a central reality of distributed computing.  
> A central problem in distributed computing is insuring that the state of the whole system is consistent after such a failure.  
> Partial failure requires that programs deal with indeterminacy.  
> Being robust in the face of partial failure requires some expression at the interface level.  
> Different implementations of an interface may provide different levels of reliability, scalability, or performance.  
> Robustness is not simply a function of the implementations of the interfaces that make up the system.  

论文还说到，NFS有两种mount方式：soft mount、hard mount。soft mount把network、server failure暴露给客户端程序，由客户端控制。hard mount则是由服务端执行挂载动作，期间客户端是hang住的，直到服务端操作完成。hard mount看起来很霸道，用户体验也不好，但却是用的最多的方式。

“hard mount用得多”告诉我们，在分布式计算中，能少一事绝不多一事，约定好过程序处理。

###Amazon

> Somewhat about the technology but more interesting is the culture and organization they've created to work with it.

1、[A Conversation with Werner Vogels](http://queue.acm.org/detail.cfm?id=1142065) - Coverage of Amazon's transition to a service-based architecture  
2、[Discipline and Focus](http://queue.acm.org/detail.cfm?id=1388773) - Additional coverage of Amazon's transition to a service-based architecture  
3、[Vogels on Scalability](http://www.itconversations.com/shows/detail1634.html)  
4、[SOA creates order out of chaos @ Amazon](http://searchwebservices.techtarget.com/originalContent/0,289142,sid26_gci1195702,00.html)  

###Google

分布式系统中的“rocket science”。

1、[MapReduce](http://labs.google.com/papers/mapreduce.html)  
2、[Chubby Lock Manager](http://labs.google.com/papers/chubby.html)  
3、[Google File System](http://labs.google.com/papers/gfs.html)  
4、[BigTable](http://labs.google.com/papers/bigtable.html)  
5、[Data Management for Internet-Scale Single-Sign-On](http://www.usenix.org/event/worlds06/tech/prelim_papers/perl/perl.pdf)  
6、[Dremel: Interactive Analysis of Web-Scale Datasets](http://www.google.com/research/pubs/pub36632.html)  
7、[Large-scale Incremental Processing Using Distributed Transactions and Notifications](http://www.google.com/research/pubs/pub36726.html)  
8、[Megastore: Providing Scalable, Highly Available Storage for Interactive Services](http://www.cidrdb.org/cidr2011/Papers/CIDR11_Paper32.pdf) - Smart design for low latency Paxos implementation across datacentres.  

###Consistency Models

> Key to building systems that suit their environments is finding the right tradeoff between consistency and availability.

1、[CAP Conjecture](http://lpd.epfl.ch/sgilbert/pubs/BrewersConjecture-SigAct.pdf) - Consistency, Availability, Parition Tolerance cannot all be satisfied at once

> Two years later, in 2002, Seth Gilbert and Nancy Lynch of MIT, formally proved Brewer to be correct and thus Brewer's Theorem was born.

2、[Consistency, Availability, and Convergence](http://www.cs.utexas.edu/users/princem/papers/cac-tr.pdf) - Proves the upper bound for consistency possible in a typical system

>=2010的一篇新论文。看不懂！

3、[Brewer's CAP Theorem](http://www.julianbrowne.com/article/viewer/brewers-cap-theorem) - Julian Browne

一篇博文，通俗地描述了CAP。

4、[Consistency and Availability](http://www.infoq.com/news/2008/01/consistency-vs-availability) - Vogels  
5、[Eventual Consistency](http://www.allthingsdistributed.com/2007/12/eventually_consistent.html) - Vogels  
6、[Avoiding Two-Phase Commit](http://www.addsimplicity.com/adding_simplicity_an_engi/2006/12/avoiding_two_ph.html) - Two phase commit avoidance approaches  
7、[2PC or not 2PC, Wherefore Art Thou XA?](http://www.addsimplicity.com/adding_simplicity_an_engi/2006/12/2pc_or_not_2pc_.html) - Two phase commit isn't a silver bullet  
8、[Life Beyond Distributed Transactions](https://database.cs.wisc.edu/cidr/cidr2007/papers/cidr07p15.pdf) - Helland  
9、[Starbucks doesn't do two phase commit](http://www.enterpriseintegrationpatterns.com/docs/IEEE_Software_Design_2PC.pdf) - Asynchronous mechanisms at work  

10、[You Can't Sacrifice Partition Tolerance](http://codahale.com/you-cant-sacrifice-partition-tolerance/) - Additional CAP commentary  

这篇文章很有意思，他很有观点：不能放弃Partition Tolerance，如果放弃了，意味着你的分布式程序必须运行在一个绝对稳定的网络中，

> For a distributed (i.e., multi-node) system to not require partition-tolerance it would have to run on a network which is **guaranteed** to never drop messages (or even deliver them late) and whose nodes are guaranteed to never die.
> 
> You and I do not work with these types of systems because **they don’t exist**.

所以，要么选择consistency，而放弃一定的availablity，或者相反，但P不能放弃。

11、[Optimistic Replication](http://www.ece.cmu.edu/~ece845/docs/optimistic-data-rep.pdf) - Relaxed consistency approaches for data replication

###Theory

> Papers that describe various important elements of distributed systems design.

1、[Distributed Computing Economics](http://research.microsoft.com/research/pubs/view.aspx?tr_id=655) - Jim Gray

计算是要花钱的，这篇来自美刀公司的Technical Report为你算算这笔帐。

2、[Rules of Thumb in Data Engineering](http://research.microsoft.com/pubs/68636/ms_tr_99_100_rules_of_thumb_in_data_engineering.pdf) - Jim Gray and Prashant Shenoy

同样来自美刀的2000年的technical report，告诉你数据会很多，会很重要...

**3、[Fallacies of Distributed Computing](http://en.wikipedia.org/wiki/Fallacies_of_Distributed_Computing) - Peter Deutsch**

Sun司大拿们总结的，分布式计算中的谬误：  
1）网络是可靠的。。。  
2）latency is zero。。。  
3）带宽是无限的。。。  
4）网络是安全的。。。  
5）拓扑结构是不会变的。。。  
6）只有一个管理员。。。  
7）transport cost is zero...  
8）网络是对等的（homogeneous：均匀的，同种的）。。。  

**4、[Impossibility of distributed consensus with one faulty process](http://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) - also known as FLP**

看标题，distributed consensus，分布式如何达成共识，是基本问题之一。却是1985年的论文，ACM的。我不由想起Zookeeper和Google Chubby。

这篇文章主要给出一个论点：没有靠谱的一致性协议。

> THEOREM 1. No consensus protocol is totally correct in spite of one fault.  
> THEOREM 2. There is a partially correct consensus protocol in which all nonfaulty processes always reach a decision, provided no processes die during its execution and a strict majority of the processes are alive initially.

结论：fault-tolerant cooperative computing cannot be solved in a totally asynchronous model of computation. 但这并不是说，分布式共识就一定搞不定，而是需要更精确的模型，比如对计算环境更多的假设、更多的预设限制（给出，我就是只能保证，在这些情况下不出大问题，如果发生其他例外，听天由命吧。。。）。

目前对此文的理解就是这样。

5、[Unreliable Failure Detectors for Reliable Distributed Systems](http://www.ecommons.cornell.edu/bitstream/1813/7192/1/95-1535.pdf). A method for handling the challenges of FLP

91年ACM上的老文，给出(4)提出问题的一个解决方案。很长，51页，共提出45+条定理和推论，尼玛，定理生产机器啊，不知道是不是我朝的刷榜风格...直接看不下去啊有木有...

6、[Lamport Clocks](http://research.microsoft.com/en-us/um/people/lamport/pubs/time-clocks.pdf) - How do you establish a global view of time when each computer's clock is independent

[Leslie Lamport](http://en.wikipedia.org/wiki/Leslie_Lamport)，便是写出Paxos算法的那个牛人。Paxos算法面世颇有一番“传奇”，这个相信很多人都懂的，于是，这篇78年的爷爷论文，估计不会啰嗦。

不过，本文要解决的问题是，分布式环境下，给event排时序。算法貌似好理解，但我想不到应用场景。

**7、[The Byzantine Generals Problem](http://research.microsoft.com/en-us/um/people/lamport/pubs/byz.pdf)**

拜占庭将军问题，直接略过论文，看其他资料：[wiki](http://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98)、[阅微堂](http://zhiqiang.org/blog/science/computer-science/tcs-byzantine-failure-the-byzantine-generals-problem.html)。

> 拜占庭将军问题 (Byzantine Generals Problem)，是由莱斯利兰伯特提出的点对点通信中的基本问题。 在分布式计算上，不同的计算机透过讯息交换，尝试达成共识；但有时候，系统上协调计算机 (Coordinator / Commander) 或成员计算机 (Member / Lieutanent) 可能因系统错误并交换错的讯息，导致影响最终的系统一致性。拜占庭将军问题就根据错误计算机的数量，寻找可能的解决办法 (但无法找到一个绝对的答案，只可以用来验证一个机制的有效程度)。

> 军队与军队之间分隔很远，传讯息的信差可能在途中路上阵亡，或因军队距离，不能在得到消息后即时回复，发送方也无法确认消息确实丢失的情形，导致不可能达到一致性。**在分布式计算上，试图在异步系统和不可靠的通道上达到一致性是不可能的**。因此对一致性的研究一般假设信道是可靠的，或不存在异步系统上而行。

8、[Lazy Replication: Exploiting the Semantics of Distributed Services](http://citeseer.nj.nec.com/ladin90lazy.html)

9、[Scalable Agreement - Towards Ordering as a Service](http://static.usenix.org/event/hotdep10/tech/full_papers/Kapritsos.pdf)

Paxos算法一般用来解决分布式共识问题，但是这些协议是不可扩展的，因而可能最终成为性能瓶颈。这篇论文提出一个新的可扩展的分布式共识方案。>=2010年，米国某大学+yahoo出品。

###Languages and Tools

1、[Programming Distributed Erlang Applications: Pitfalls and Recipes](http://docmanual.com/Read/_vp.bWFuLmx1cGF3b3JsZC5jb20-_vp..sl_content.sl_develop.sl_p37-svensson.pdf) - Building reliable distributed applications isn't as simple as merely choosing Erlang and OTP.

###Infrastructure

1、[Principles of Robust Timing over the Internet](http://queue.acm.org/detail.cfm?id=1773943) - Managing clocks is essential for even basics such as debugging

###Storage

1、[Consistent Hashing and Random Trees](http://www.akamai.com/dl/technical_publications/ConsistenHashingandRandomTreesDistributedCachingprotocolsforrelievingHotSpotsontheworldwideweb.pdf)，distributed caching protocols for relieving Hot Spots on the world wide web  
2、[Amazon's Dynamo Storage Service](http://www.allthingsdistributed.com/2007/10/amazons_dynamo.html)

###Paxos Consensus

1、[The Part-Time Parliament](http://research.microsoft.com/users/lamport/pubs/lamport-paxos.pdf) - Leslie Lamport  

**2、[Paxos Made Simple](http://research.microsoft.com/users/lamport/pubs/paxos-simple.pdf) - Leslie Lamport**  

Paxos解决的问题，不同于拜占庭将军问题，虽然可能有一条消息被传递多次，但绝不会出现错误的消息。

[libpaxos](http://libpaxos.sourceforge.net/)

3、[Paxos Made Live - An Engineering Perspective](http://labs.google.com/papers/paxos_made_live.html) - Chandra et al  
4、[Revisiting the Paxos Algorithm](http://groups.csail.mit.edu/tds/paxos.html) - Lynch et al  
5、[How to build a highly available system with consensus](http://research.microsoft.com/lampson/58-Consensus/Acrobat.pdf) - Butler Lampson  
6、[Reconfiguring a State Machine](http://research.microsoft.com/en-us/um/people/lamport/pubs/reconfiguration-tutorial.pdf) - Lamport et al - changing cluster membership  
7、[Implementing Fault-Tolerant Services Using the State Machine Approach: a Tutorial](http://citeseer.ist.psu.edu/viewdoc/summary?doi=10.1.1.20.4762) - Fred Schneider  

###Other Consensus Papers

1、[Mencius: Building Efficient Replicated State Machines for WANs](http://www.usenix.org/event/osdi08/tech/full_papers/mao/mao_html/) - consensus algorithm for wide-area network

###Gossip Protocols(Epidemic Behaviours)

一类模拟社会中流言传播方式的协议？

> ...The concept of gossip communication can be illustrated by the analogy of office workers spreading rumors...[[Gossip protocol](http://en.wikipedia.org/wiki/Gossip_protocol)]

1、[How robust are gossip-based communication protocols?](http://infoscience.epfl.ch/record/109302?ln=en)  
2、[Astrolabe: A Robust and Scalable Technology For Distributed Systems Monitoring, Management, and Data Mining](http://www.cs.cornell.edu/home/rvr/papers/astrolabe.pdf)  
3、[Epidemic Computing at Cornell](http://www.allthingsdistributed.com/historical/archives/000456.html)  
4、[Fighting Fire With Fire: Using Randomized Gossip To Combat Stochastic Scalability Limits](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.5.4000)  
5、[Bi-Modal Multicast](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.17.7959)  

###Experience at MySpace

###eBay