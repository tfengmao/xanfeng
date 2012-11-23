---
title: 顺序锁(seqlock)和RCU
layout: post
category: kernel
tags: seqlock rcu
---

顺序锁(seqlock)很好理解，RCU相反，RCU后面好像[隐藏着很深的知识](http://www.rdrop.com/users/paulmck/rclock/RCU.TU-Dresden.2012.05.15a.pdf)。  

###seqlock(顺序锁)

seqlock的内容转载自kouu的“[linux seqlock & rcu浅析](http://hi.baidu.com/_kouu/item/0b99dae513c2b4b52f140bd1)”。  

在linux内核中，有很多同步机制。比较经典的有spin_lock（忙等待的锁）、mutex（互斥锁）、semaphore（信号量）、等。并且它们几乎都有对应的rw_XXX（读写锁），以便在能够区分读与写的情况下，让读操作相互不互斥（读写、写写依然互斥）。  
而seqlock和rcu应该可以不算在经典之列，它们是两种比较有意思的同步机制。  

seqlock用于能够区分读与写的场合，并且是读操作很多、写操作很少，写操作的优先权大于读操作。  
seqlock的实现思路是，用一个递增的整型数表示sequence。写操作进入临界区时，sequence++；退出临界区时，sequence再++。写操作还需要获得一个锁（比如mutex），这个锁仅用于写写互斥，以保证同一时间最多只有一个正在进行的写操作。  
当sequence为奇数时，表示有写操作正在进行，这时读操作要进入临界区需要等待，直到sequence变为偶数。读操作进入临界区时，需要记录下当前sequence的值，等它退出临界区的时候用记录的sequence与当前sequence做比较，不相等则表示在读操作进入临界区期间发生了写操作，这时候读操作读到的东西是无效的，需要返回重试。  

seqlock写写是必须要互斥的。但是seqlock的应用场景本身就是读多写少的情况，写冲突的概率是很低的。所以这里的写写互斥基本上不会有什么性能损失。  
而读写操作是不需要互斥的。seqlock的应用场景是写操作优先于读操作，对于写操作来说，几乎是没有阻塞的（除非发生写写冲突这一小概率事件），只需要做sequence++这一附加动作。而读操作也不需要阻塞，只是当发现读写冲突时需要retry。  

seqlock的一个典型应用是时钟的更新，系统中每1毫秒会有一个时钟中断，相应的中断处理程序会更新时钟（见《linux时钟浅析》）（写操作）。而用户程序可以调用gettimeofday之类的系统调用来获取当前时间（读操作）。在这种情况下，使用seqlock可以避免过多的gettimeofday系统调用把中断处理程序给阻塞了（如果使用读写锁，而不用seqlock的话就会这样）。中断处理程序总是优先的，而如果gettimeofday系统调用与之冲突了，那用户程序多等等也无妨。

seqlock的实现非常简单：  
{% highlight c %}
// 写操作进入临界区时：  
void write_seqlock(seqlock_t *sl)
{
	spin_lock(&sl->lock); // 上写写互斥锁
	++sl->sequence;       // sequence++
}
// 写操作退出临界区时：
void write_sequnlock(seqlock_t *sl)
{
	sl->sequence++;         // sequence再++
	spin_unlock(&sl->lock); // 释放写写互斥锁
}

// 读操作进入临界区时：
unsigned read_seqbegin(const seqlock_t *sl)
{
	unsigned ret;
repeat:
	ret = sl->sequence;      // 读sequence值
	if (unlikely(ret & 1))   // 如果sequence为奇数自旋等待
		goto repeat;
	return ret;
}
// 读操作尝试退出临界区时：
int read_seqretry(const seqlock_t *sl, unsigned start)
{
	return (sl->sequence != start); // 看看sequence与进入临界区时是否发生过改变
}
// 而读操作一般会这样进行：
do {
	seq = read_seqbegin(&seq_lock);      // 进入临界区
		do_something();
} while (read_seqretry(&seq_lock, seq)); // 尝试退出临界区，存在冲突则重试
{% endhighlight %}

###RCU

RCU不容易被理解，且看作者给出的资料：[Introduction to RCU](http://www.rdrop.com/users/paulmck/rclock/).  
作者是Paul E. McKenney，他今年的讲稿"[What Is RCU?](http://www.rdrop.com/users/paulmck/rclock/RCU.TU-Dresden.2012.05.15a.pdf)"看起来非常有料！  

RCU也适合读多写少的场合，和reader-writer lock（rwlock）相比，其优势是：更好的性能，死锁免疫，更好的实时性，读端可并发。  
这里只列出“[Introduction to RCU](http://www.rdrop.com/users/paulmck/rclock/)”资料中的图片：  
![](/images/rcu-list-replace.jpg)  
\[list_replace_rcu\]  
![](/images/rcu-grace-period.png)  
\[grace period，根据fs/lockd/grace.c的注释，“is a period during which locks should not be given out”\]  
![](/images/rcu-rwlock-compare.jpg)  
\[RCU和rwlock的比较\]  