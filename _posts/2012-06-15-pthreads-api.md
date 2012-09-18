---
title: pthreads--userspace perspective
layout: post
tags: pthreads userspace mutex thread schedule
category: linux
---

*在"[more pthreads](http://xanpeng.github.com/2012/05/15/more-linux-pthreads/)"之前, 我就已经"围攻"pthreads了, 然后在这篇文章中, 我发愿要弄懂pthreads. 之前, 我都是从内核层去尝试理解的, 今天是要从用户态使用的角度来看pthreads. 主要参考了书籍<POSIX多线程程序设计>.*

创建线程, 使用互斥量(mutex)这些都是routine, 不多说, 使用时查看reference即可, 比如<POSIX thread API concept>, "[POSIX Threads Programming](https://computing.llnl.gov/tutorials/pthreads/)".  
本文关注一些更有意思的话题(*因为没有代码验证, 可能有错, 虽验证*).

---

#condition不可或缺?

mutex用来控制对数据的访问, 条件变量condition是用来通知线程数据状态变化的.  

**使用mutex的时候, 是否必须同时使用condition?**  

不是的. 看下面的例子: 启动4个线程去计算两个浮点序列的乘积之和, 每个线程计算一部分乘积和, 最后需要把和累加到一个共享的变量dotstr.sum中. 在这个例子中, 就没有使用condition.

<script src="https://gist.github.com/2940066.js"> </script>

这个例子中用到的就是mutex最根本的功能: 访问保护对象之前, 调用mutex_lock, 加锁成功则获得保护对象的访问权限, 加锁失败则暂时不能获得权限, 后面某时会获得权限.  
这里的问题是: mutex_lock加锁失败时, 线程定然是被阻塞而不能继续执行, 那么线程能否被再次调度? 线程怎么检测他人何时释放锁? 是通过循环检测式的"忙等"吗?  
stackoverflow有类似的讨论, 分别从[协议](http://stackoverflow.com/a/5267931/264035)和[manpage](http://stackoverflow.com/a/5267865/264035)方面给出答案, *但都是语义层次的答案, 并不是实现机制层次的答案. 我也暂时不纠结其后面的细节, 因为语义变动少, 而实现机制可以依平台变化而变.* 答案是: mutex_lock加锁失败后被blocked, 直到某时获得锁, 如果有多个mutex_lock在等待, mutex_unlock时根据scheduling policy选择哪一个mutex_lock获得锁.

那么, **condition又用在什么时候**?  

前面mutex的例子中, 线程直接去操作共享的sum变量, 不需要检查sum的现有状态. 考虑另一种情况: 一个处理队列的线程A如果发现队列为空, 它只能等待, 直到有新的数据被插入队列. 队列是多个线程共享的, 因而由一个mutex保护, 线程A必须锁住mutex才能判定队列的当前状态, 如果判定队列为空, 线程A将要等待, 不过在等待之前必须释放锁, 否则其他线程就不能插入数据. 线程A在等待之后, 就不能再知道队列的状态了, 即使线程B已经往队列中插入数据, 于是线程A只有永远等待下去. **这就是需要condition的原因**.

实际上也可以这么做, 线程A锁定mutex, 判断队列为空, 将自己加入到队列的wait_list中, 然后释放mutex. 线程B锁定mutex, 往队列中插入数据, 检查到wait_list中有A, 于是释放mutex, 并通知线程A队列非空. 线程A锁定mutex, 判定队列真的非空, 将自己移出wait_list, 然后处理.  
这么做似乎是可行的, 虽然没有经过严格的论证. 但这么做的缺点是需要自己维护一个wait_list, 增加编程复杂度和出错概率. 不过, 或许condition的实现机制就是类似的方案呢.

condition是与mutex相关, 也与mutex保护的共享数据相关的信号机制. 等待condition总是返回锁住的mutex, condition相关的API是和mutex"纠缠"在一起的, 这是因为它们"搞基"的天性决定的(-_-|), 看一个例子: 两个线程更新count, 第三个线程等待count达到某个特定值.

<script src="https://gist.github.com/2940207.js"> </script>

---

#线程私有数据

*<POSX多线程程序设计>这个话题的观感很差啊, 我是看"[Threads Primer](http://www8.cs.umu.se/kurser/TDBC64/VT03/pthreads/pthread-primer.pdf)"的.*

线程私有数据(Thread Specified Data, TSD)的实现机制依vendor不同而不同, 但一般来说都大同小异. 至于TSD是什么, 为什么需要有TSD, 就不再多说, 是线程内做到函数间共享数据的方式. 

pthreads通过一组API实现TSD, 大致思路是, 每个线程维护一个表格, 表格中放TSD数据, 定义key表示表格内的offset. 如下图所示.

*待插入图片8.1*

使用pthreads TSD, 先创建一个key, key被加入到所有线程可见的全局TSD数组中. 然后使用pthread_getspecific()和pthread_setspecific()访问key关联的数据. 一般这么使用TSD.

    pthread_key_t house_key;
    
    foo((void *) arg) {
        pthread_setspecific(house_key, arg);
        bar();
    }

    bar() {
        float n = (float) pthread_getspecific(house_key);
    }

    main() {
        ...
        pthread_keycreate(&house_key, destoryer);
        pthread_create(&tid, NULL, foo, (void*) 1.414);
        pthread_create(&tid, NULL, foo, (void*) 3.14159);
        ...
    }

pthreads的TSD实现对于用户来说有些不便使用的, 而且让初学者如我有些迷惑, 实际上可以认为它使用一个静态的map关联线程和TSD数据的, 如[这篇文章](http://stackoverflow.com/questions/8988253/thread-specific-data-why-cant-i-just-use-a-static-map-with-thread-ids)讨论的. C++11提供了更好的API.

---

#线程和核实体

#死锁检测

#实时调度



