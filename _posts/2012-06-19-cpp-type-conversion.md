---
title: c++ type conversion/casting
layout: post
tags: type c++ conversion cast
category: coding
---

*md, 不写c++代码很多年了, 所以, 谨慎对待本文所说, 你懂的*.  
本文主要参考cplusplus上面的"[Type Casting](http://www.cplusplus.com/doc/tutorial/typecasting/)".

c++是一种强类型语言, 所谓"强", 在我理解便是每个数据都有其类型, 操作时严格注意类型间是否兼容. 对应的"弱", python是否算作一种, 据我"早年"有限的python经验, python可以先让var指向一个链表, 后面又可以让它指向一个整数, 且无需在赋值前指定var的类型. ---- 不管对错, 以及后面的根源, "强弱"非本文重点.

c++的类型体系没有一个"root", 众所周知, Java所有类型都是Object的子类. c++的类型分基本类型和用户自定义类型两种. 基本类型有int, char, 指针类型等, 用户自定义类型就是自定义的class了.

如果同时操作不同类型的变量, 则可能面对类型转换问题. 如将一个short变量赋值给一个int变量.  
类型转换分两大类: 隐式的转换(implicit)和显式的转换(explicit). 隐式的转换是那种显然符合语义的转换, 比如short转到int; 而显式转换则是需要程序员明确告诉编译器: 我这么做是经过考虑的.

还是从基本类型和类类型两方面来看type casting.

**基本类型的type casting**

对于基本类型来说, 隐式转换就不用说了, 无非就是"从小到大"转是安全的. 我们熟悉的显式转换方式有两种:  
- c-like cast notation. 如"int b = (int) a;".  
- functional notation. 如"int b = int(a)".  
如果这么做, 我们能直白地知道是否可行. 但对于class, 再使用这种形式的type casting, 如果有问题, 就不能那么直白地被发觉了.

需要特别指出, 指针属于基本类型, 可以对它使用上面这两种转换方式. 但是如果将指针和class type结合起来, 使用不得当可能就会出现问题, 而且可能不易于发现. 下面阐述.

**class type的type casting**  

class type casting也有隐式显式之分.  

先说隐式. 实际上, 隐式转换就是调用ctor而已, 如"B b = a;", 相当于调用ctor B(A a), 则隐式转换成立的前提就是你必须提供这样的ctor.  
实际上, 到了class type, 很多内容是与ctor连系起来的. [基本类型是没有ctor的](http://stackoverflow.com/questions/5113365/do-built-in-types-have-default-constructors).

再说class type的显式转换. 先说旧风格, "B b = (B)a;"形式是不能通过编译的, "B* pb = (B*)&a"形式是可以通过编译的, 但结果就是UB了, 在我看来, 它可以是显然是因为用了指针这一基本类型.

class type中显示转换最重要的是那些convert operator: static_cast, const_cast, dynamic_cast, reinterpret_cast.  
下面分别说明这四种casting oprator.  
- dynamic_cast. 只能被用于指针和引用, 目的是保证To是From的一个合法, 完整的对象. *意思是从object layout来看, 要求To是From的一部分?*  
- static_cast. 理解不深刻, 不多说了. 反正是可以在父子类的指针间乱转的.  
- reinterpret_cast. 也不多说了. 类似于强转的意思, 给你一个地址, 随便你怎么去解释内容.  
- const_cast. 也不多说了. 好像是用于去除const属性的.

**implicit_cast**

上面草草结束了*_cast的分析, 此处要来看为什么要有implicit_cast.

    template<typename To, typename From>
    inline To implicit_cast(From const &f) {
        return f;
    }
    
    // 用法举例
    size_t avail_size = implicit_cast<size_t>(avail_());

"[What is the difference between static_cast and Implicit_cast?](http://stackoverflow.com/questions/868306/what-is-the-difference-between-static-cast-and-implicit-cast)"中讨论了此话题. static_cast可以做down-cast, down-cast是隐式转换(*区别implicit_cast, 这是一个非官方的cast operator*)不允许的, implicit_cast跟隐式转换一样, 不允许down-cast, 所以它safer.

*疲了, 先水完, 后续会有update.*




