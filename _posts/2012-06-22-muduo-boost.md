---
title: boost in muduo
layout: post
tags: boost muduo cpp
category: programming
---

本次借了解"[muduo's c++ high-perf logging](http://xanpeng.github.com/2012/06/18/muduo-logging/)"学习c++, 本文介绍其中使用的boost.  

#ptr_vector

muduo AsyncLogging使用ptr_vector来管理buffer的. ptr_vector是[ptr_container](http://www.boost.org/doc/libs/1_49_0/libs/ptr_container/doc/ptr_container.html)的一种, 后者代表pointer container library.  

这么来理解ptr_container:  
1. 创建很多Foo对象, 用std::vector<Foo>.  
2. 用std::vector<Foo*>.  
3. 用std::vector<boost::shared_ptr<Foo> >.  
4. 用ptr_vector<Foo>.  

ptr_container和std::vector<shared_ptr<Foo> >相比的优劣可以看[这里](http://www.boost.org/doc/libs/1_49_0/libs/ptr_container/doc/ptr_container.html#motivation).  

**[shared_ptr](http://www.boost.org/doc/libs/1_49_0/libs/smart_ptr/shared_ptr.htm)**又用来做什么? 它是smart pointer的一种, 用来管理对象的使用, 确保在无人使用时执行dtor. [这里](http://www.boost.org/doc/libs/1_49_0/libs/smart_ptr/example/shared_ptr_example.cpp)有一个示例程序:  
- 其中涉及**[for_each](http://www.cplusplus.com/reference/algorithm/for_each/)**, 这是一个十分好用的函数.  
- 还涉及了函数对象(Function Object), 实际上是调用操作符(call operator). 据<c++ primer v4> 14.8 "call operator and function objects": 定义了call operator的类对象被称为function objects, 因为它们被定义为类, 但使用方式就像调用函数一样, 见如下代码. -- *曾经让我纠结的Function Object原来如此简单:)*.

    struct FooPtrOps {        <---- 定义处
        bool operator()(const FooPtr & a, const FooPtr & b) { return a->x > b->x; }
        void operator()(const FooPtr & a) { std::cout << a->x << "\n"; }
    };
    ...
    std::for_each(foo_vector.begin(), foo_vector.end(), FooPtrOps()); <---- 调用处

shared_ptr体现的[pimpl idiom](http://www.boost.org/doc/libs/1_49_0/libs/smart_ptr/shared_ptr.htm#Handle/Body)当然也是值得一提的.
    

