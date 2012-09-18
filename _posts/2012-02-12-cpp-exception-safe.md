---
title: C++ 异常安全
layout: post
tags: c++ exception
category: programming
---

异常安全  
　　basic guarantee: 无资源泄露.  
　　strong guarantee: 事务级别.  
　　参考 [Exception-safe class design](http://www.gotw.ca/gotw/059.htm)  
　　　　 [copy-and-swap idioms](http://en.wikibooks.org/wiki/More_C++_Idioms/Copy-and-swap)

Nothrow guarantee  
　　保证某些函数是不会抛异常的. 这是实现 strong guarantee 的保障. 

exception-safe assignment  
　　使用 copy-and-swap idom, 先创建临时对象, 再调用 nothrow 的 swap 函数.  
　　　　Type temp(s);  
　　　　swap(this, temp);


