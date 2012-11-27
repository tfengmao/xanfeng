---
title: elf和elftoolchain
layout: post
category: linux
tags: elf libelf readelf objdump x86_64 amd64 ia-64
---

我在看pthreads  
-> 了解进程组和线程组  
-> 了解进程组的ppid、pid  
-> 了解getpid系统调用  
-> 想通过ltrace查看`ps`调用的库函数，从而找到getpid在glibc中的入口  
-> 编译ltrace需要libelf支持  
-> 放狗找libelf时看到[elftoolchain](http://sourceforge.net/apps/trac/elftoolchain/)

elftoolchiain维护了一个列表，把ld，[readelf](http://en.wikipedia.org/wiki/GNU_Binutils)，[objdump](http://en.wikipedia.org/wiki/Objdump)，libelf等都包含进去了.

> We maintain BSD licensed implementations of essential compilation tools and libraries for handling ELF based program images.

###libelf by example

[libelf-by-example](http://mdsp.googlecode.com/files/libelf-by-example-20100112.pdf)是很不错的资料，它介绍了libelf的使用和elf layout。具体这里不说了，某时再去看原文档吧(一定抽空;-))。仅列出文档中的几个例子：  

<script src="https://gist.github.com/2717224.js"> </script>

Example: Reading an ELF executable header.  
Example: Reading a Program Header Table.  
Example: Listing section names.  
Example: Creating an ELF object.  
Example: Stepping through an ar(1) archive.  

###elf standards

elf是一个协议，它定义了一种文件格式，诸多工具按照这个格式写入或读出。对于64bit OS，不同的厂商的CPU（如Intel [IA-64](http://en.wikipedia.org/wiki/IA-64#Architecture)和AMD [AMD64](http://en.wikipedia.org/wiki/AMD64#AMD64)）是否会对elf有特别的规定呢？[Application Binary Interface](http://en.wikipedia.org/wiki/Application_binary_interface)？还不了解，也暂不继续 follow 了。

[ELF(Executable and Linkable Format)](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)  
[ELF-64 Object File Format](http://downloads.openwatcom.org/ftp/devel/docs/elf-64-gen.pdf)  
[Generic System V Application Binary Interface](http://www.sco.com/developers/gabi/latest/contents.html)  
[Generic System V Application Binary Interface Edition4.1](http://www.sco.com/developers/devspecs/gabi41.pdf)  
[AMD64 System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf)  

![](/images/elf-format.png)

###更多资料
  
[How To Write Shared Libraries by Ulrich Drepper](http://www.akkadia.org/drepper/dsohowto.pdf)  
[LibElf and GElf - A Library to Manipulate ELf Files](http://developers.sun.com/solaris/articles/elf.html)  
[ELF: From The Programmer's Perspective](http://linux4u.jinr.ru/usoft/WWW/www_debian.org/Documentation/elf/elf.html)  
[Linkers and Loaders](http://linker.iecc.com/)  