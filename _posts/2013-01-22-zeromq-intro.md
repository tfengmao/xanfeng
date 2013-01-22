---
title: ZeroMQ浅析
layout: post
category: coding
tags: network pattern zeromq zmq
---

我摘引过一篇官方介绍zeromq的文章“Fix the World”，zeromq（希望）抽象出网络编程的基本单元，然后在编写大型网络项目时，只要像堆lego积木一样，利用基本单元，方便地“组装”。  

zeromq有很多值得学习的东西：1）它对网络编程模式的抽象，和各种经验；2）它对C++的应用，以及作者受不了C++ exception机制的“不成熟”，转而开始用C重写，这个转换过程中的讨论；3）各种语言binding的实现；4）网络编程的关键细节：可靠性、丢包、重连等；5）作者对于编程的理解；6）...  

zeromq有三个方面的资料：  
1、官方文档zguides和wiki。zguides很受推崇，即使你不想深入zeromq和网络编程，它仍然很值得一读。作者在其中融入了他的编程经验和理念，比如：  

> There are **three main open source patterns**. One, the large firm dumping code to break the market for others. This is the Apache Foundation model. Two, tiny teams or small firms building their dream. This is the most common open source model, and it can be very successful commercially. Three, aggressive and diverse communities that swarm over a problem landscape. This is the Linux model, and the one we aspire to with ØMQ.

2、源代码。从作者反思C++实现的“恶果”，可以感觉的出来，作者对代码的精雕细琢。初看来，我认为这是一份功能上“完美”、性能上“完美”并且外表也“完美”的代码。  
3、中文社区的讨论。主要是博文，但可惜有价值的不多，大家都是浅尝辄止。云风的“[ZeroMQ的模式](http://blog.codingnow.com/2011/02/zeromq_message_patterns.html)”最值得推荐，因为我也认为，学习zeromq的最好入口、最好目的就是网络编程的模式。

在网络编程方面，我日常接触很少，现在位于“Hello World”水平，因此本文也很遗憾地不能说的深入。  
仅列出我粗读zguides后，整理出来的zeromq骨架，留待后续补充。  

###3/4 basic patterns

**Exclusive pair**(一对一结对通信), should use only to connect two threads in a process.  
**Requst-Reply**: connect a set of clients to a set of services.This is a remote procedure call and task distribution pattern.  
**Publish-subscribe**: a data distribution pattern.  
**Pipeline**: connects node in a fan-out / fan-in pattern, a parallel task distribution and collection pattern.  

###valid zmq::socket_t combinations

- PUB-SUB
- REQ-REP
- REQ-ROUTER
- DEALER-REP
- DEALER-ROUTER
- DEALER-DEALER
- ROUTER-ROUTER
- PUSH-PULL
- PAIR-PAIR

XPUB, XSUB like raw version of PUB, SUB.

###advanced patterns  

【1】Intermediaries, or **proxies**, **queues**, forwarders, **device**, **brokers**   
Pub-Sub Network with a proxy:  
![](https://github.com/imatix/zguide/raw/master/images/fig13.png)  

Request-reply Broker:   
![](https://github.com/imatix/zguide/raw/master/images/fig17.png)  

Pub-Sub Forwarder Proxy:  
![](https://github.com/imatix/zguide/raw/master/images/fig18.png)

【2】Pub-Sub Synchronization
![](https://github.com/imatix/zguide/raw/master/images/fig22.png)

【3】Signaling between Threads  
![](https://github.com/imatix/zguide/raw/master/images/fig21.png)

【4】Load-balancing Pattern  
a Load-Balancing Broker:   
![](https://github.com/imatix/zguide/raw/master/images/fig32.png)

【5】Asynchronous Client-Server Pattern  
![](https://github.com/imatix/zguide/raw/master/images/fig37.png)

![](https://github.com/imatix/zguide/raw/master/images/fig38.png) 

【6】[A group of Reliable Request-Reply Patterns](http://zguide.zeromq.org/page:all#Chapter-Reliable-Request-Reply-Patterns)  
The Lazy Pirate Pattern:  
![](https://github.com/imatix/zguide/raw/master/images/fig47.png)

The Simple Pirate Pattern:  
![](https://github.com/imatix/zguide/raw/master/images/fig48.png)

The Paranoid Pirate Pattern:  
![](https://github.com/imatix/zguide/raw/master/images/fig49.png)

Service-Oriented Reliable Queuing(The Majordomo Pattern):  
![](https://github.com/imatix/zguide/raw/master/images/fig50.png)  

Disconnected Reliability(Titanic Pattern):  
![](https://github.com/imatix/zguide/raw/master/images/fig51.png)

High-availability Pair(Binary Star Pattern):  
![](https://github.com/imatix/zguide/raw/master/images/fig52.png)  
![](https://github.com/imatix/zguide/raw/master/images/fig53.png)  
[Preventing Split-Brain Syndrome](http://zguide.zeromq.org/page:all#Preventing-Split-Brain-Syndrome)  

Brokerless Reliability(Freelance Pattern):  
![](https://github.com/imatix/zguide/raw/master/images/fig55.png)  

【7】A group of Advanced Publish-Subscribe Patterns  
[Pub-sub tracing (Espresso Pattern)](http://zguide.zeromq.org/page:all#Pub-sub-Tracing-Espresso-Pattern)  

[Slow Subscriber Detection (Suicidal Snail Pattern)](http://zguide.zeromq.org/page:all#Slow-Subscriber-Detection-Suicidal-Snail-Pattern)  

High-speed Subscribers(Black Box Pattern):  
![](https://github.com/imatix/zguide/raw/master/images/fig56.png)  
![](https://github.com/imatix/zguide/raw/master/images/fig57.png)  

[Reliable Publish-Subscribe(Clone Pattern)](http://zguide.zeromq.org/page:all#Reliable-Publish-Subscribe-Clone-Pattern)

###other details

High water marks