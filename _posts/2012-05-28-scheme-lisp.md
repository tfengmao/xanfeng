---
title: Scheme/Lisp做什么
layout: post
tags: scheme lisp language
category: coding
---

王垠在"[初学者程序语言的选择](http://blog.sina.com.cn/s/blog_5d90e82f01015271.html)"中向初学者推荐 [schemer](http://www.schemers.org/)(*就是scheme, 之所以去掉'r', 是因为某编辑器/编译器曾经存在的字符长度限制*), 我好奇为什么不推荐c?   
加上我也久仰[函數程式語言](http://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B8%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80)(*[functional programming](http://en.wikipedia.org/wiki/Functional_programming)*), 好奇c到底是不是函数程式语言呢?  

经由上述两点驱动, 我这两天抽空阅读了 "The Little Schemer"(*TLS*) 和 "[SICP](http://groups.csail.mit.edu/mac/classes/6.001/abelson-sussman-lectures/)" 的chapter 1.  
TLS 读起来非常轻松, 作者讲解的非常明白, 如它很多章节的标题一样, "again, and again, and again, ...", scheme语法非常简单. 我没有了解其所有细节, 就其主要方面, scheme完全就像**搭积木**一样, 所以它能用来做什么, 就看你能想到什么, 所以复杂的部件不过是由简单的部件堆积而已, 堆积, 再堆积 -- again, and again...

看了SICP的chapter 1和第一课的视频之后, 给我同样的感觉. 未经深思: 平时看来高深的 **lambda** 的概念, 现在觉得很直白, 就是定义函数而已, 顶多只是 online define.  
-- *我的这个理解多是小白了...*

于是, 得出 scheme/lisp 的**优点**之一: 极其简单的 primitives 和语法, 却又极其强大.

但在了解更多之间, 我又有**担忧**: scheme 的应用面广泛否, 我不曾经历过一个即使很小的实际应用!  
scheme 不流行的原因是什么呢?  
我能想到的原因之一: scheme 写出的程式的确太"**难看**", 在括号间, 我很容易迷失(*或许也因为被c/c++等形式的语言先入为主吧*). 这绝对是阻止我的最大原因.  

关于 scheme/lisp 的讨论:  
1. [Lisp's reputation is so bad that many people don't even take a look at Lisp](http://ilc2009.scheming.org/node/6).  
2. [Are there people using scheme out there?](http://stackoverflow.com/questions/291033/are-there-people-using-scheme-out-there)  
3. [Learning Lisp - Why?](http://stackoverflow.com/questions/4724/learning-lisp-why)  
4. [Does anyone use the Scheme programming language for a living?](http://stackoverflow.com/questions/1521544/does-anyone-use-the-scheme-programming-language-for-a-living)  

搜索得知, 我略能理解的 scheme/lisp 用的比较多的地方有: 教育界(*比如用来做编程语言研究, 用来试验新语言的特性*), AI 和航空航天(*历史原因+冰河介绍*), 另外其descriptive特性, 或许可以用作不错的脚本语言吧.  

对我来说, schemer/lisp 的了解暂不会继续, 我目前不能领悟其美妙之处. 相反地它的"阅读不便利+是否有充分的积木库"让我胸中积郁.  
so, that's it.

more:  
[Lisp Success Stories](http://bc.tech.coop/blog/041027.html) 列出了很多 Lisp 做的不错的领域.  
[Lisp 中文社区Wiki](http://lisp.org.cn/wiki/)