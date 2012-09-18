---
title: abi of compiler
layout: post
tags: abi compiler cpp
category: programming
---

*2012-06-30 更新*

在"[binary compatibility in c++](http://xanpeng.github.com/2012/06/29/cpp-abi)"中, 我提到C++的二进制兼容性, 提到C++的ABI, 并且提到compiler ABI和操作系统ABI, 本文来讨论compiler ABI.

那篇文章通过对比"binary compatibility"和"source compatibility"来理解二进制兼容性, 本文来对比API和ABI.  
API大家很熟悉, C++在头文件中描述对外开发的函数的格式, 这就是API. 而ABI跟前篇文章描述的一样, 描述了compiler在编译时所作的特殊处理, 比如使用的alignment和layout等.

实际上, 我认为G++的compiler abi就是前篇文章描述的C++ abi, 或者C++是没有abi一说的, 编译器才有abi一说. gnu的文档"[ABI policy and guidelines](http://gcc.gnu.org/onlinedocs/libstdc++/manual/abi.html)"给出了一个式子:

> "library API + compiler ABI = library ABI"

一个插曲: G++的不同版本之间ABI规范是有变更的, 于是使用不同版本G++编译出的library和程序可能不能组合使用. 有一个[ABIcheck](http://abicheck.sourceforge.net/)工具可做这方面的检查.

我在giantchen的"[二进制兼容性](http://www.cnblogs.com/Solstice/archive/2011/03/09/1978024.html)"博文作出评论:

> 请问, 本文描述的C++ ABI应该指的就是编译器ABI吧? 比如对于G++来说, 要遵循某个ABI. 而就C++本身是没有ABI一说的吧?

> 对于G++ ABI(我参阅这个链接: http://gcc.gnu.org/onlinedocs/libstdc++/manual/abi.html), 我是这么理解的: 输入是C++源码, 处理者是G++, 输出是elf文件, elf是有格式标准的, G++的输出必须遵循这个标准, 但此标准却留有自由度, 对于此自由度, 不同的compiler, 以及同一compiler的不同版本可以有不同的处理方式(也就是所谓convention), 这个特定处理便是 ABI 所关注的了.

giantchen回复: "理解错误. 这里讲的是库的ABI."  
又出来一个"库ABI"...但我暂不深究, 我想遇到实际问题时, 我应能知道为什么, 虽然我不知道可将问题划归违背"C++ ABI", "G++ ABI"或者"库ABI".
