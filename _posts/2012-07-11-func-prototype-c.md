---
title: C语言函数原型的重要性
layout: post
category: coding
tags: function prototype c
---

本文涉及:  
1. 隐式声明的危害.  
2. gcc的built-in函数.  

这里说的函数原型是function prototype, 也就是函数声明, 是程序告诉gcc函数样貌的一种方式.  
我们知道, 如果在使用函数之前, 没有提供原型声明, 那么gcc只好根据调用掉吗做隐式声明.  
**隐式声明**是靠不住的, 因为是由gcc根据调用推导出来的, 因而可能和程序员意图不符合. 并且, 最重要地, gcc默认返回值是int类型, 这样可能有严重的后果. 看下面的例子:

{% highlight cpp %}
#include <errno.h>
#include <stdio.h>
 
int main(int argc, char *argv[]) {
	FILE *fp;
 
	fp = fopen(argv[1], "r");
	if (fp == NULL) {
		fprintf(stderr, "%s\n", strerror(errno));
		return errno;
	}
 
	printf("file exist\n");
    
	fclose(fp);
	return 0;
}
{% endhighlight %}

不使用和使用`-Wall`编译选项, 以及执行的结果如下:

{% highlight text %}
# make test
cc     test.c   -o test

# gcc -Wall test.c -o test
test.c: In function ‘main’:
test.c:9: warning: implicit declaration of function ‘strerror’
test.c:9: warning: format ‘%s’ expects type ‘char *’, but argument 3 has type ‘int’

# ./test
Segmentation fault
{% endhighlight %}

可以看出, strerror()的隐式声明返回值被认为是int, 但实际上strerror返回的是一个字符串的地址, 在64位机器上, 地址值大小为8个字节, int值大小是4个字节(*x86_64是LP-64模型*), 所以strerror()返回时地址值被截断, 产生一个invalid的地址值, 于是必然segfualt.  
解决办法很简单, 通过`#include <string.h>`给出显式声明即可.

再看另一个例子, 同样是没有提供函数原型声明, 只不过这次是malloc()和free().  

{% highlight cpp %}
#include <stdio.h>

int main(void) {
	int *p = malloc(sizeof(int));
    
	if (p == NULL) {
		perror("malloc()");
		return -1;
	}

	*p = 10;
	free(p);
	return 0;
}
{% endhighlight %}

根据上面的思路, 照理说, malloc()在返回时, 真正的地址值要被截断, 然后赋给指针p, 这个值是非法的, 那么下面更改*p就会segfault. 但实际情况并非如此, 至于为什么, 还不得而知.  
不过, 这次编译的warning有些特别:

{% highlight text %}
# gcc -o test test.c
test.c: In function ‘main’:
test.c:5: warning: incompatible implicit declaration of built-in function ‘malloc’
test.c:13: warning: incompatible implicit declaration of built-in function ‘free’

# gcc -Wall -o test test.c
test.c: In function ‘main’:
test.c:5: warning: implicit declaration of function ‘malloc’
test.c:5: warning: incompatible implicit declaration of built-in function ‘malloc’
test.c:13: warning: implicit declaration of function ‘free’
test.c:13: warning: incompatible implicit declaration of built-in function ‘free’

# gcc test.c -o test -Wall -fno-builtin
test.c: In function ‘main’:
test.c:5: warning: implicit declaration of function ‘malloc’
test.c:5: warning: initialization makes pointer from integer without a cast
test.c:13: warning: implicit declaration of function ‘free’
{% endhighlight %}

亮点在于warning中的"built-in function", gcc的built-in函数是一些在编译期间就被计算完成的辅助函数, 比如知名的likely()和unlikely(), 它们在编译期间就被替换为特定的处理方式.  
[gcc文档](http://gcc.gnu.org/onlinedocs/gcc-4.0.3/gcc/Other-Builtins.html)给出了built-in函数列表. 也可以使用`-fno-builtin`取消使用.