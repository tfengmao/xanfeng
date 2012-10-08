---
title: 实模式和保护模式
layout: post
tags: linux real protected mode
category: linux
---

在学习 linux 内存管理的时候, 看到说 linux 启动时首先运行在实模式下, 随后转到保护模式下. 于是 google 了一下实模式和保护模式.  
实模式和保护模式都是对 intel 处理器而言的<sup>[nix][]</sup>.

**实模式(real mode)**  
x86 处理器在启动的时候, 首先运行于实模式. 此时处理器的行为就像 8086, 它仅能访问有限的一组处理器指令, 并且实模式下处理器仅能访问 1MB 的内存<sup>[wb][]</sup>.

**保护模式(protected mode)**  
保护模式下, 处理器能访问 >1MB 的内存空间. 保护模式下系统提供了更多的功能: 多任务, 内存保护, 分页系统等. 很多 x86 操作系统都运行于保护模式, 如 Linux, FreeBSD 和 Microsoft Windows 的某些版本<sup>[nix][]</sup>.

[nix]: http://nixcraft.com/linux-hardware/6343-real-mode-protected-mode.html "nixcraft"
[wb]: http://en.wikibooks.org/wiki/X86_Assembly/Protected_Mode "wikibooks"

