---
title: memory allocation in c++
layout: post
tags: memory allocation cpp new allocator operator placement delete override
category: programming
---

*2012-06-29 更新.*

侯捷的[<池内春秋: Memory Pool的设计哲学和无痛运用>](http://jjhou.boolan.com/programmer-13-memory-pool.pdf)介绍了C++平台提供的内存分配工具, 分别是:  
- malloc/free.  
- new/delete.  
- ::operator new()/::operator delete().  
- alloc::allocate()/alloc::deallocate().  

"池内春秋"着重介绍了C++ SGI STL的allocator, 也就是vector<int, allocator<int> >的第二个模板参数. 下文成SGI STL allocator为alloc, alloc使用的算法大致为: 首要思想是bulk allocation, 不是每次都往system heap申请内存. 建立memory pool和free lists两个概念, free lists一般是16个, 以8为单位递增, 这些空闲链表分别是9,16,24,...,128, (*这个想法类似于Linux物理内存管理中的slab分配器*). 小于128字节的内存请求都在free lists中分配, 大于128字节的都使用malloc()分配. 一开始free lists为空, 内存请求来临的时候, alloc向memory pool申请, memory pool刚开始可能也为空, 它向system heap申请. 申请时采用一定的方法, 不必细说, 可以想象其目的.

alloc使用这种bulk allocation策略, 提升大量用户内存请求的效率.  
这篇"[Overloading operator new](http://www.relisoft.com/book/tech/9new.html)"则细致地介绍了如何重载global new和delete, 如何重载class-specific new.

陈硕的"[C++ 工程实践(2)：不要重载全局 ::operator new()](http://www.cnblogs.com/Solstice/archive/2011/02/22/1960301.html)"则表示, 尽量**不要**在实际工程中重载global new(), 因为会带来库与库之间交流的麻烦, 实在有重载global new()的需求, 也可以用malloc()替代.   
陈硕的"[C++ 标准库中的allocator是多余的](http://blog.csdn.net/Solstice/article/details/4401382)"表示STL的allocator是**多余**的, 但我并不能完全理解此文, 谨记之. 不过在我所有的toy程序中, 并没有使用特殊allocator的需求.

---

是否重载global new, stackoverflow上也有讨论, 我基本上认同giantchen的观点. 至于他认为allocator是多余的, 我心有戚戚, 继续了解更多一点allocator.

[Allocator(C++)](http://en.wikipedia.org/wiki/Allocator_(C%2B%2B))是Alexander Stepanov设计的, 此人据myan[一篇博文](http://blog.csdn.net/myan/article/details/5928531)提及, "是一个披着人皮的外星智慧". 在C++标准引入STL的过程中, 标准委员会认为对memory model的完全抽象会引起unacceptable的性能损失, 为了补救, "the requirements of allocators were made more restrictive. As a result, the level of customization provided by allocators is more limited than was originally envisioned by Stepanov."

**standard(default) C++ STL allocator是如何被使用的?**  
drddobbs上一篇文章"[The Standard Librarian: What Are Allocators Good For?](http://www.drdobbs.com/the-standard-librarian-what-are-allocato/184403759)"指出, allocators是STL中最神秘的一个部分, 它很少被显式地调用. 此文认为: 除非要实现自己的STL container, 否则你是不需要调用allocator的.  
allocator不同于new, 这在前文介绍"池内春秋"时也提到. 至于default allocator是如何申请内存的, 是用malloc, 还是其他, 则要看其实现了.

**如何用customized allocator替换default allocator?**  
存在需要custom allocator的场景, 一般都是希望获得更好的性能, 处理特别的memory model, 或者跟踪内存访问等. 侯捷的<The Annotated STL Source>开篇便讲述了allocator的细节, 以及如何去customize allocator.  

另, csdn上也有[对allocator的讨论](http://topic.csdn.net/t/20020426/10/677768.html), myan参与其中, 他的发言非常有助于理解, 摘引如下:

> 我不知道你自己是否动手做过一些工具类。我有过这方面的体验，因为原来WinCE下没有STL，所以一开始设计一个类，中间想用到String类的时候，就自己去写；想用到vector，list的时候，就自己写个简单的。结果发现，不论想写什么东西，一定会碰到动态内存管理的问题，而且最基本的new/delete一般都不够理想。因此设计一个通用的内存管理组件，是始终萦绕在心头的想法。有了allocator，其他各个组件可以将内存管理的策略交给allocator来处理，自己保持一种无知状态，则复杂度大大降低。而且可以选择不同的内存管理策略，灵活应对不同的需求。 
> 
> lattice说的很对，你如果想了解allocator，目前能够看到最好的解释是侯先生的TASS。那本书里将allocator放在所有组件之前讲述，这一安排真的是体现了侯先生的功力。

> SGI STL的allocator完全脱离了标准STL的规范，你写的allocator越是符合标准，越无法与SGI   STL的allocator相容。这部分内容，等到侯捷先生的《STL源码剖析》发行之后，你一看便知。 
> 
> 至于allocator是否是解决动态内存分配问题的良好手段，我仍然觉得在STL里，是个好方案。你给出的方案，实际上就是通过重载new/delete，自己管理内存。这种方案比allocator来得直接，一般来说，越直接越有效，越好移植。但是放在STL这个框架里，这是不行的。 

---

了解allocator如上, 可知STL allocator用在为container动态分配空间服务, 前文提到giantchen认为C++ STL allocator是多余的, "...C++的allocator是依赖注入的一次失败的尝试", 我目前持怀疑态度, 但我对其文提及的相关话题理解有限, 因而对allocator的理解有待更新.