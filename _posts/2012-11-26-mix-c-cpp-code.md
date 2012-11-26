---
title: 混用C、C++代码
layout: post
category: coding
tags: c cpp extern link
---

这个话题网上会有很多资料。这里我根据C++ FAQ “[How to mix C and C++](http://www.parashift.com/c++-faq-lite/mixing-c-and-cpp.html)”整理一遍。  
C++程序可以使用C代码，C程序也可以使用C++代码。本文侧重C++程序使用C代码。  

C++程序和C程序的区别主要体现在两处：调用规范（calling convention）和名字处理（name mangling）。  
C++程序编译出来的程序，函数名和变量名都是经过处理的，比如class AAA变成\_ZTI3AAA。然后再被linker连接。  
而C程序则完全“原汁原味”，名字没有经过处理，test()编译出来仍然是test。  

所以要在C++程序中使用编译好的C库，就要解决这两个问题，主要是name mangling问题。  
有两种方法：1、C++程序员多做事情；2、C程序员多做事情。  
1、C++程序员多做事情。  
{% highlight cpp %}
extern "C" {
	// Get declaration for f(int i, char c, float x)
	#include "my-C-code.h"
}
// 或者如果只用到f()，可以只声明这一个函数
extern "C" void f(int i, char c, float x);
// 或者用到一些函数
extern "C" {
	void f(int i, char c, float x);
	int g(char* s, char const* s2);
	double sqrtOfSumOfSquares(double a, double b);
}

int main()
{
	f(7, 'x', 3.14);   // Note: nothing unusual in the call
	...
}
{% endhighlight %}

2、C程序员多做事情（推荐）。  
{% highlight c %}
// 在C头文件开头增加（当然，不一定是绝对开头，这里表意即可）
#ifdef __cplusplus
extern "C" {
#endif

// 在C头文件结尾增加
#ifdef __cplusplus
}
#endif

// C++程序这么使用
// Get declaration for f(int i, char c, float x)
#include "my-C-code.h"   // Note: nothing unusual in #include line

int main()
{
	f(7, 'x', 3.14);       // Note: nothing unusual in the call
  	...
}
{% endhighlight %}

其他：  
1、使用C标准库函数时，C++程序员不需要另做动作。  
2、C、C++程序[最好使用同一个厂家的编译器](http://www.parashift.com/c++-faq-lite/overview-mixing-langs.html)。  
