---
title: c/c++ calling conventions
layout: post
tags: cdecl stdcall convention
category: programming
---

cdecl: 函数参数压栈顺序 "从右到左", 调用者负责清理堆栈.  
stdcall: 函数压栈顺序 "从右到左", 被调用者负责清理堆栈.  

x86 calling conventions:  
* http://en.wikipedia.org/wiki/X86_calling_conventions

stdcall & cdecl:  
* http://stackoverflow.com/questions/3404372/stdcall-and-cdecl

Calling conventions, for different C++ compilers and operating systems:  
* http://www.agner.org/optimize/calling_conventions.pdf

The history of calling conventions:  
* http://blogs.msdn.com/b/oldnewthing/archive/2004/01/08/48616.aspx

