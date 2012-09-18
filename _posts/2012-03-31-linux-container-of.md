---
title: Linux 中的 container_of 宏
layout: post
tags: linux container_of macro offset typeof
---

Linux源码中很多地方用到container_of这个宏，定义位于linux/kernel.h，内容：

    #define **container_of**(ptr, type, member) ({			\
	    const **typeof**( ((type *)0)->member ) *__mptr = (ptr);	\
	    (type *)( (char *)__mptr - **offsetof**(type,member) );})

这个宏的**作用**: 根据一个指向结构体成员的指针ptr，获取结构体type的指针。

很多网文分析了这个宏，比如[这个](http://senghoo.com/229.html)就很不错。  
其原理很简单：  
- 第一行的typeof操作符是GCC编译器指定的，获取操作数的类型。  
- 第一行的((type *)0)->member是一个小技巧，搭配typeof可以定位到结构成员member的类型。  
- 第一行定义__mptr指针的意图：有人猜测是保存ptr，以防止外部被更改，我觉得另一个原因是利用**类型检查**。  
- 第二行表示：将ptr减去member在结构体中的偏移，就得到了结构体的指针了。  
- 第二行的offsetof是另一个宏，定义位于linux/stddef.h，内容：

    #ifdef __compiler_offsetof
    #define offsetof(TYPE,MEMBER) __compiler_offsetof(TYPE,MEMBER)
    #else
    #define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
    #endif

（接上）要么使用编译器的__compiler_offsetof（这个此处不管），要么使用和第一行一样的技巧：((type \*)0)->member，*如果结构体从0地址开始，那么member的地址就是它在结构体中的偏移了*。

container_of这个宏可以利用到用户态的程序中：
    
    #include <stdio.h>
    
    #define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
    
    #define container_of(ptr, type, member) ({          \
        const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
        (type *)( (char *)__mptr - offsetof(type,member) );})
    
    struct xtype
    {
        int a;
        int b;
        char c;
    };
    
    int main()
    {
        struct xtype x;
        x.a = 10;
        x.b = 100;
        x.c = 'x';
    
        int* pb = &x.b;
        struct xtype *px = container_of(pb, struct xtype, b);
        printf("%d %d %c\n", px->a, px->b, px->c);
    
        return 0;
    }
