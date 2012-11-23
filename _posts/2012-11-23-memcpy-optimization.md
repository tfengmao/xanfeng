---
title: memcpy的优化
layout: post
category: coding
tags: cacheline memcpy prefetch alignment compiler
---

在weibo上看到一段描述memcpy的话：  
> @Stanley文威：memcpy可是软件优化和CPU架构相结合的典型例子。WORD对齐，cacheline对齐，预取等等都在里面，我记得我有好几个同事优化memcpy优化了好久。属于计算机里的微雕艺术。  
> @vbvan：有人还专门写了一本书。  
可惜我没有找到这本书。  

【[memcpy performance](http://software.intel.com/en-us/articles/memcpy-performance)】  
memcpy的原型：`void *memcpy (void *restrict __dest, const void *restrict __src, size_t __n)`，编译器优化是由const、restrict支持的。  
[restrict](http://en.wikipedia.org/wiki/Restrict)是C99提出的一个关键词，用来修饰指针ptr，含义是ptr指向的内容，除了通过ptr，或ptr+1之类的方式访问，不会通过其他方式访问，比如不会定义新指针去指向。这是对编译器的一个“提示”，如果使用restrict之后，编程时又不遵守此“提示”，则会有Undefined Behaviour。  
> The restrict keyword is a declaration of intent given by the programmer to the compiler.  
> This limits the effects of pointer aliasing, **aiding caching optimizations**. If the declaration of intent is not followed and the object is accessed by an independent pointer, this will result in undefined behavior.   
*gcc编译包含restrict关键词的程序时，要使用`--std=c99`.*  

另外，32-bit gcc对memcpy做了激进的(aggressive)优化，比如如果拷贝常数大小内存，则直接将代码内联了。  
*gcc选项`-mmemcpy/-mno-memcpy`可用来控制这个细节。*  

【云风 “[VC 对memcpy的优化](http://blog.codingnow.com/2005/10/vc_memcpy.html)”】  
