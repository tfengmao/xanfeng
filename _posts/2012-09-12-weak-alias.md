---
title: 弱符号(weak symbol)和别名(alias)
layout: post
category: coding
tags: glibc socket weak symbol alias
---

###weak symbol

弱符号(weak symbol)是gcc的特性，比如给foo加上\_\_attribute\_\_((weak))，foo实际上可以不存在，这样也不阻止程序正确编译链接。  
> Weak dynamic linking is another feature that can be used to tell the dynamic linker to ignore missing symbols. 

{% highlight cpp %}
// foo.c 
// gcc -Wall -fPIC -c foo.c
// gcc -shared -o libfoo.so foo.o
extern void foo(void) __attribute__((weak));
void fun(void) {
    if (foo) foo();
}

/////////////////////////////////////
// weak_test.c
// gcc weak_test.c -o weak_test -lfoo
#include <stdio.h>

void fun(void);

/*
void foo(void) {
    printf("Hello\n");
}
*/

int main() {
    fun();
    return 0;
}
{% endhighlight %}

上面的代码中，在weak_test.c中即使不定义foo，程序也能正确编译链接执行。如果定义了foo，打印“Hello”。  

至于linker处理弱符号的细节，没去了解。看起来ld在链接时，会做一个检查，如果没有foo，也不报错。

###weak_alias

gcc还提供一个特性：weak_alias，它为一个符号定义一个别名，直接看例子：  
{% highlight cpp %}
// bar.c
// gcc -Wall -c -fPIC bar.c
// gcc -shared bar.o -o libbar.so
#include <stdio.h>

void _bar(void) {
    printf("_weak_bar()\n");
}

// 实现1
#  define weak_alias(name, aliasname) _weak_alias (name, aliasname)
#  define _weak_alias(name, aliasname) \
      extern __typeof (name) aliasname __attribute__ ((weak, alias (#name)));
weak_alias(_bar, bar)

// 实现2
// void bar() __attribute__((weak, alias("_bar")));

///////////////////////////////////////////
// weakalias.c
// gcc -Wall weakalias.c -lbar -o weakalias
#include <stdio.h>

/*
void bar(void) {
    printf("do real bar()\n");
}
*/

int main() {
    bar();
    return 0;
}
{% endhighlight %}

bar.c中写了两种weak alias的实现，方式一就是glibc的做法，它实际上等价与方式二。  
在这个例子中，bar是\_bar的别名，main里面调用bar的时候，实际上就是调用\_bar.  
weak在这里的含义：bar这个别名是weak symbol，于是可以重新定义bar，就像在weakalias.c中注释的代码那样。

glibc多处用到weak_alias，比如socket函数定义为__socket的别名，weak_alias(\_\_socket, socket)，但\_\_socket几乎是一个空函数，显然不是我们需要的socket函数逻辑。  
实际的socket函数是以汇编代码的方式，重新定义在glibc/sys-deps/unix/sysv/linux/i386/socket.S中的。

###资源

1、王聪的[强符号，弱符号](http://wangcong.org/blog/archives/262)  
2、[Fun with weak dynamic linking](http://glandium.org/blog/?p=2764)  