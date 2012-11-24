---
title: elf和elftoolchain
layout: post
category: linux
tags: elf libelf readelf objdump x86_64 amd64 ia-64
---

#calltrace

我在看 pthreads  
-> 了解进程组和线程组  
-> 了解进程组的 ppid 和 pid  
-> 了解 getpid 系统调用  
-> 想通过 ltrace 查看 ps 调用的库函数, 从而找到 getpid 在 glibc 中的入口  
-> 编译 ltrace 需要 libelf 支持  
-> 放狗找 libelf 时看到 [elftoolchain](http://sourceforge.net/apps/trac/elftoolchain/)

elftoolchiain 维护了一个列表, 把 ld, [readelf](http://en.wikipedia.org/wiki/GNU_Binutils), [objdump](http://en.wikipedia.org/wiki/Objdump), libelf 等都包含进去了.

> We maintain BSD licensed implementations of essential compilation tools and libraries for handling ELF based program images.

#libelf by example

[libelf-by-example](http://mdsp.googlecode.com/files/libelf-by-example-20100112.pdf) 是很不错的资料, 它通过示例介绍了 libelf 的使用和 elf 的layout. 具体这里不说了, 某时再去看原文档吧(一定抽空;-)). 仅列出文档中的几个例子:

**hello, libelf**

<script src="https://gist.github.com/2717224.js"> </script>

Example: Reading an ELF executable header.  
Example: Reading a Program Header Table.  
Example: Listing section names.  
Example: Creating an ELF object.  
Example: Stepping through an ar(1) archive.  

---

#elf standards

可以想见, elf 是一个协议, 它定义了一种文件格式, 诸多工具按照这个格式写入或读出. 来了解一下这个格式. 对于 64bit OS, 不同的厂商的 CPU(如同属[x86_64](http://en.wikipedia.org/wiki/X86-64)体系结构的 Intel 的 [IA-64](http://en.wikipedia.org/wiki/IA-64#Architecture), 以及 AMD 的 [AMD64](http://en.wikipedia.org/wiki/AMD64#AMD64)) 是否会对 elf 有特别的规定呢? [Application Binary Interface](http://en.wikipedia.org/wiki/Application_binary_interface)? 还不了解, 也暂不继续 follow 了.

[ELF(Executable and Linkable Format)](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)  
[ELF-64 Object File Format](http://downloads.openwatcom.org/ftp/devel/docs/elf-64-gen.pdf)  
[Generic System V Application Binary Interface](http://www.sco.com/developers/gabi/latest/contents.html)  
[Generic System V Application Binary Interface Edition4.1](http://www.sco.com/developers/devspecs/gabi41.pdf)  
[AMD64 System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf)  

![](http://upload.wikimedia.org/wikipedia/commons/thumb/7/77/Elf-layout--en.svg/541px-Elf-layout--en.svg.png)

---

# Further Reading
  
[How To Write Shared Libraries by Ulrich Drepper](http://www.akkadia.org/drepper/dsohowto.pdf)  
[LibElf and GElf - A Library to Manipulate ELf Files](http://developers.sun.com/solaris/articles/elf.html)  
[ELF: From The Programmer's Perspective](http://linux4u.jinr.ru/usoft/WWW/www_debian.org/Documentation/elf/elf.html)  
[Linkers and Loaders](http://linker.iecc.com/)  