---
title: C++实现线程安全的Singleton
layout: post
category: programming
tags: cpp singleton thread
---

如何用C++实现线程安全的单例模式(singleton)，本文汇总这方面的讨论，包括DCL(double-checked-locking)、meyers singleton和采用pthread_once()的方案，并最终决定在今后**选择pthread_once()方案**。

在现在的我看来，陈硕的muduo是一个宝库，从中可以学到C++、网络编程、分布式应用、多线程服务程序等方面的实践知识。muduo不是朝夕之功，而且其背后隐藏的价值不下于代码本身，陈硕自2008年始就写作了多篇相关博文，很值得学习，比如“[多线程服务器的常用编程模型](http://files.cppblog.com/Solstice/multithreaded_server.pdf)”。此文提到了线程安全的Singleton实现，我想到去年曾了解这个话题，这次再遇，打算记录下来。

###Double-Checked Locking Pattern

DCLP不是线程安全的，因为可能有乱序执行的影响，Meyers的[这篇文章](http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf)有讨论。

{% highlight cpp %}
Sigleton* Singleton::instance() {
	if (pInstance == 0) {
		Lock lock;
		if (pInstance == 0) {
			pInstance = new Singleton;
		}
	}
	return pInstance;
}
{% endhighlight %}

###Meyers Singleton

该实现使用了lazy initialization，看起来线程安全，但实际上不是的。

{% highlight cpp %}
static Singleton& instance() {
	static Singleton s;
	return s;
}
{% endhighlight %}

因为实际上编译器类似于如此处理(看[这个链接](http://stackoverflow.com/questions/1661529/is-meyers-implementation-of-singleton-pattern-thread-safe)，里面有不错的扩展资料)：

{% highlight cpp %}
static Singleton& instance() {
    static bool initialized = false;
    static char s[sizeof( Singleton)];

    if (!initialized) {
        initialized = true;
        new( &s) Singleton(); // call placement new on s to construct it
    }

    return (*(reinterpret_cast<Singleton*>( &s)));
}
{% endhighlight %}

###pthread_once()

这个方案被stackoverflow上一些人以及陈硕支持，下面的代码来自于陈硕的博文。

{% highlight cpp %}
#include <pthread.h>

template<typename T>
class Singleton : boost::noncopyable {
public:
	static T& instance() {
		pthread_once(&ponce_, &Singleton::init);
		return *value_;
	}
	
	static void init() {
		value_ = new T();
	}
private:
	static pthread_once_t ponce_;
	static T* value_;
};

template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;

template<typename T>
T* Singleton<T>::value_ = NULL;
{% endhighlight %}