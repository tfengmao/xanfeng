---
title: 如何实现异常处理
layout: post
category: coding
tags: exception handle
---

**注意：本文理解多有不当**，尤其是C++例子中FPE根本就没有被捕获！我最新的思考位于：[理解C++异常处理](http://xanpeng.github.com/programming/2012/07/31/cpp-exception-handle.html)。

本文讨论程序语言如何实现异常处理, 从而能深刻地理解异常处理. 由于水平和经验不足, 所述多是个人理解, 欢迎交流和共同探讨.

了解一些资料之后, 发觉异常处理这个话题也很复杂. 比如对于C++, 往下说就是什么时候触发异常, 内核如何处理异常, 如何把异常信息告知程序. 往上说就是, C++ catch并处理异常之后, 是继续执行还是终止执行(*参见[wiki/Exception_handling](http://en.wikipedia.org/wiki/Exception_handling)上提到的termination semantics和resumption semantics*), 以及如何实现exception-safe的类和方法等.

内核的处理机制是固定的, 我想它提供了一些API和消息传送机制, 供用户态使用. 如C++要实现异常处理功能, 就需要用到这些API和消息机制. ULK ch4"中断和异常"说, 80x86 CPU发布了大约20种不同的异常, 而内核为每种异常提供了一个专门的处理程序. 这20种异常中, 我们熟悉的有"Divide error", "Overflow", "Segmentation Fault", "Floating point error".

异常分[软件异常和硬件异常](http://en.wikipedia.org/wiki/Exception_handling), 我想硬件异常(*以ULK ch4为准*)才是最根本的, 对所有编程语言都一致的, 不管编程语言是否处理, 它们都存在那里, 该发生时还是发生. 因此硬件异常属于系统概念. 而软件异常则属于应用层面的概念, 在不同的编程语言间不一致. 比如Java中有FileNotFoundException异常, 但实际上完全可以没有它, 比如没有异常处理的C, 程序通过检查fd判断文件是否打开成功, 据之执行不同的逻辑.

但C程序中, 如果疏于检查而使得除零发生, 此时触发硬件异常, 由于C没有异常处理机制, 程序此时就会终止. 这是一种不同的策略, C里面广泛地通过参数检查, 通过返回值检查, 通过全局errno检查, 做了很多其他语言-Java, C++等-的"普通"异常处理工作. 但是当严重的异常发生时, 比如segfault和除零, Java等是可以捕捉这些异常, 并让程序继续执行(*resumption semantics*). 但C则没有处理, C认为程序已经严重出错, 回天乏术, 不如直接terminate掉.

当这些真正的异常发生时, C是让程序终止, C++使用的是termination semantic还是resumption sematic呢? 实例表明C++使用的是termination semantics, C++采取这种方式是出于性能考虑, wiki页面有相关讨论. 而Java和Python则使用了resumption semantics(但Iowa State大学的[这篇文档](http://www.cs.iastate.edu/~tingz/classes/cs342/Fall2010/lectures/handout-17.pdf)说Java和Python也是termination semantics). 下面给出示例.

{% highlight c %}
#include <stdio.h>
int main() {
    int a = 100 / 0;
    printf("hello world\n");
    return 0;
}
{% endhighlight %}

{% highlight text %}
# ./c-exception
Floating point exception
{% endhighlight %}

{% highlight cpp %}
#include <iostream>
using namespace std;
int main() {
    try {
        // int a = 100 / 0;	// test-1
        throw 20;		// test-2
    } catch (...) {
        cout << "exception occurred." << endl;
    }
    cout << "continue after exception handling" << endl;
    return 0;
}
{% endhighlight %}

{% highlight text %}
test-1的输出:
# ./cc-exception
Floating point exception

test-2的输出:
# ./cc-exception
exception occurred.
continue after exception handling
{% endhighlight %}

{% highlight java %}
import java.io.*;
public class ExcepTest {
    public static void main(String args[]) {
        try {
            int a = 100 / 0;
        } catch (Exception e) {
            System.out.println("exception catched: " + e);
        }
        System.out.println("out of try block");
    }
}
{% endhighlight %}

{% highlight text %}
# java ExcepTest
exception catched: java.lang.ArithmeticException: / by zero
out of try block
{% endhighlight %}

**总结**  
有些编程语言提供了异常处理功能, 有些没有. 不管有没有提供, 除零这样的异常是客观存在的, 并且在内核中有对应的处理程序(*参考ULK ch4*). 提供了异常处理功能的编程语言可以定义自己的异常类型, 比如FileNotFoundException, 这样的异常只不过是一种上层的辅助策略. 这种辅助策略可以用来简化编程, 不过也会带来额外的问题, 比如[google C++项目就不推荐使用异常](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml#Exceptions).

可以考虑阅读CPython的异常处理实现, 来佐证上面的猜想.
