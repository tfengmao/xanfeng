---
title: 序列点(sequence point)
layout: post
category: coding
tags: c cpp
---

[sequence point](http://en.wikipedia.org/wiki/Sequence_point)就像是程序语句执行的barrier一样，执行到sequence point的时候，之前的语句的所有side effect都已经执行过了。根据sequence point可以确定程序某些语句的执行顺序，不能根据sequence point确定顺序的语句，其执行的效果就可能是未定的了。  
摘引wikipedia的定义：  
> A sequence point in imperative programming defines any point in a computer program's execution at which it is guaranteed that all side effects of previous evaluations will have been performed, and no side effects from subsequent evaluations have yet been performed.  
> They are **often mentioned in reference to C and C++**, because the result of some expressions can depend on the order of evaluation of their subexpressions.

c/c++中的sequence point有<sup>[wiki](http://en.wikipedia.org/wiki/Sequence_point)</sup>：  
> * Between evaluation of the left and right operands of:
>	* && (logical AND), 
>	* || (logical OR),
>	* comma operators, 
>	* a ? b : c. // at the "question-mark" operator
> * Before a function is entered in a function call(whether or not the function is inline).
>	* Note that a function call f(a,b,c) is not a use of the comma operator and the order of evaluation for a, b, and c is unspecified.
> * At a function return, after the return value is copied into the calling context.
> * At the end of a full expression.
> * At the end of an initializer.
> * Between each declarator in each declarator sequence; for example, between the two evaluations of a++ in int x = a++, y = a++;

stackoverflow上["Undefined Behavior and Sequence Points"](http://stackoverflow.com/questions/4176328/undefined-behavior-and-sequence-points)也讨论了这个问题：  
> * RULE 1: Between the previous and next sequence point a scalar object shall have its stored value modified *at most once* by the evaluation of an expression.
> * RULE 2: Furthermore, the prior value shall be accessed only to determine the value to be stored.
>	* It means if an object is written to within a full expression, any and all accesses to it within the same expression **must be directly involved** in the computation of the value to be written. e.x.: `i = i + 1` is fine. （在同一个表达式中，如果在多个地方访问同一变量，那么这些访问的目的必须是 (a)都只是读；(b)都为了计算一个新值。——很明显，这个限定的目的是防止这样的场景：在同一个表达式中，我读你写；我到底是先读，还是你先写？我读到的值完全依赖于这个顺序了...）  
>	* `std::printf("%d %d", i, ++i);` invokes Undefined Behaviour.

看一个示例，下面程序的输出是什么？  
{% highlight c %}
#include <stdio.h>

void func(int val)
{
    printf("val: %d\n", val);
}

int main(int argc, char *argv[]) 
{
    int a = 1;
    func(a++);	// 怎么理解这个a++？
    return 0;
}
{% endhighlight %}