---
title: zeromq：建造世界
layout: post
category: coding
tags: zeromq socket
---

看zeromq guide的时候，首章“[Fixing the World](http://zguide.zeromq.org/page:all#Fixing-the-World)”描述了软件世界，它的说法与我的认知符合，故简译于此。

How to explain ØMQ? Some of us start by saying all the wonderful things it does. It's sockets on steroids. It's like mailboxes with routing. It's fast! Others try to share their moment of enlightenment, that zap-pow-kaboom satori paradigm-shift moment when it all became obvious. **Things just become simpler. Complexity goes away. It opens the mind.** Others try to explain by comparison. It's smaller, simpler, but still looks familiar. Personally, I like to remember why we made ØMQ at all, because that's most likely where you, the reader, still are today.  
但事物变简单的时候，或者经过我们的努力（封装），使得事物使用变得简单的时候，我们便更容易打开思路，构建更丰富的上层功能。  

Programming is a science dressed up as art, because most of us don't understand the physics of software, and it's rarely if ever taught. The physics of software is not algorithms, data structures, languages and abstractions. These are just tools we make, use, throw away. The real physics of software is the physics of people.  
软件有如人类，软件的世界有如人类世界。人们编写软件，视之为工具，构造更好的人类世界。  

Specifically, our limitations when it comes to complexity, and our desire to work together to solve large problems in pieces. This is the science of programming: make building blocks that people can **understand and use easily**, and people will work together to solve the very largest problems.  
通信是必然需求，软件之间也要“社交”，软件通过通信构建出更大型的软件。因而基础设施必须简单易用。  

We live in a connected world, and modern software has to **navigate** this world. So the building blocks for tomorrow's very largest solutions **are connected and massively parallel**. It's not enough for code to be "strong and silent" any more. **Code has to talk to code**. Code has to be chatty, sociable, well-connected. Code has to run like the human brain, trillions of individual neurons firing off messages to each other, a massively parallel network with no central control, no single point of failure, yet able to solve immensely difficult problems. And it's no accident that the future of code looks like the human brain, because the endpoints of every network are, at some level, human brains.  
未来的软件世界必然充满通信和交互。  

If you've done any work with threads, protocols, or networks, you'll realize this is pretty much impossible. It's a dream. Even connecting a few programs across a few sockets is plain nasty, when you start to handle real life situations. Trillions? The cost would be unimaginable. Connecting computers is so difficult that software and services to do this is a multi-billion dollar business.

So we live in a world where the wiring is years ahead of our ability to use it. We had a software crisis in the 1980s, when leading software engineers like Fred Brooks believed there was no "Silver Bullet" to "promise even one order of magnitude of improvement in productivity, reliability, or simplicity".

Brooks missed free and open source software, which solved that crisis, enabling us to share knowledge efficiently. Today we face another software crisis, but it's one we don't talk about much. Only the largest, richest firms can afford to create connected applications. There is a cloud, but it's proprietary. Our data, our knowledge is disappearing from our personal computers into clouds that we cannot access, cannot compete with. Who owns our social networks? It is like the mainframe-PC revolution in reverse.

We can leave the political philosophy for [another book](http://swsi.info/). The point is that while the Internet offers the potential of massively connected code, the reality is that this is out of reach for most of us, and so, large interesting problems (in health, education, economics, transport, and so on) remain unsolved because there is no way to connect the code, and thus no way to connect the brains that could work together to solve these problems.

There have been many attempts to solve the challenge of connected software. There are thousands of IETF specifications, each solving part of the puzzle. For application developers, HTTP is perhaps the one solution to have been simple enough to work, but it arguably makes the problem worse, by encouraging developers and architects to think in terms of big servers and thin, stupid clients.

...