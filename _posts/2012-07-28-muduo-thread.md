---
title: 理解muduo.thread
layout: post
category: coding
tags: pthread muduo condition mutex
---

迄今为止，我发布的muduo相关文章有：  
1、[boost in muduo](http://xanpeng.github.com/programming/2012/06/22/muduo-boost.html)   
2、[muduo's c++ high-perf logging](http://xanpeng.github.com/programming/2012/06/18/muduo-logging.html)  
迄今为止，我发布的pthreads相关文章有：  
1、[Linux pthreads 多线程库](http://xanpeng.github.com/linux/2012/03/28/linux-pthread.html)  
2、[more pthreads](http://xanpeng.github.com/linux/2012/05/15/more-linux-pthreads.html)  
3、[pthreads--userspace perspective](http://xanpeng.github.com/linux/2012/06/15/pthreads-api.html)  

自夸：可以看出认知愈来愈深，文章写作也愈来愈好。现在看来，前文有两大缺点：1、内容不足不深，表述不清。2、自7.9换用新博客模板，旧文布局多有瑕疵。  
这是遗憾，也是必然。

muduo.thread指代muduo中的C++多线程库，陈硕的“[发布一个 Linux 下的 C++ 多线程库](http://blog.csdn.net/Solstice/article/details/5829421)”做出简介。  
库的内容：  
1、整数的原子操作，AtomicInt32、AtomicInt64  
2、线程，Thread  
3、线程池，ThreadPool  
4、互斥量和条件变量，MutexLock、Condition、MutexLockGuard  
5、模仿Java concurrent的BlockingQueue和CountdownLatch  
6、Singleton和ThreadLocal  

###AtomicInt32，AtomicInt64

定义于Atomic.h中，分别是模板类AtomicIntegerT的特化，

> typedef detail::AtomicIntegerT<int32_tAtomicInt32;  
> typedef detail::AtomicIntegerT<int64_tAtomicInt64;

我平时很少接触到使用AtomicInt的需求，不理解其使用场景。其原子性是通过[gcc atomic-builtin](http://gcc.gnu.org/onlinedocs/gcc-4.1.1/gcc/Atomic-Builtins.html)方法实现的，如：

{% highlight cpp %}
T get() {
    return __sync_val_compare_and_swap(&value_, 0, 0);
}
T getAndAdd(T x) {
    return __sync_fetch_and_add(&value_, x);
}
{% endhighlight %}

###Condition、MutexLock和MutexLockGuard

muduo用于Linux环境，故而MutexLock、Condition等必然是pthreads的抽象，使之适合C++面向对象语义，使具更优的表达能力。要理解Mutex等，需要理解pthreads，并理解如何抽象才更为合适。

####mutex

pthreads中的mutex：  
1、变量类型pthread_mutex_t。  
2、初始化，有两种方式：静态PTHREAD_MUTEX_INITIALIZER，动态pthread_mutex_init()。  
3、加锁pthread_mutex_lock()，pthread_mutex_trylock()用的较少。  
4、解锁pthread_mutex_unlock()。  
5、销毁pthread_mutex_destroy()。  
6、attr相关操作。  

MutexLock封装pthreads mutex，ctor初始化mutex，dtor销毁mutex，lock()封装pthread_mutex_lock()，unlock则相反。同时，MutexLock还记录mutex的holder。看代码（去除不重要的细节，改变代码排版）：

{% highlight cpp %}
class MutexLock : boost::noncopyable {
public:
    MutexLock() : holder_(0) { pthread_mutex_init(&mutex_, NULL); }
    ~MutexLock() {
		assert(holder_ == 0);
        pthread_mutex_destroy(&mutex_);
    }

    bool isLockedByThisThread() { return holder_ == CurrentThread::tid(); }
    void assertLocked() { assert(isLockedByThisThread()); }

    // internal usage
    void lock() {
        pthread_mutex_lock(&mutex_);
        holder_ = CurrentThread::tid();
    }
    void unlock() {
        holder_ = 0;
        pthread_mutex_unlock(&mutex_);
    }

private:
    pthread_mutex_t mutex_;
    pid_t holder_;
};
{% endhighlight %}

MutexLockGuard通过ctor和dtor封装Mutex lock和unlock的调用，避免用户直接调用lock/unlock，预防错误使用。代码很简单：

{% highlight cpp %}
class MutexLockGuard : boost::noncopyable {
public:
    explicit MutexLockGuard(MutexLock& mutex) : mutex_(mutex) {
        mutex_.lock();
    }
    ~MutexLockGuard() { mutex_.unlock(); }
private:
    MutexLock& mutex_;
};
{% endhighlight %}

####condition

pthreads中条件变量condition的使用是和mutex天生关联的，多个线程访问共享内存，就某些特定条件需要做出协调时，就需要使用condition。如对公共队列queue，producer和consumer都访问它，producer往其中push，consumer从其中pop，则producer应等待queue非满，consumer需等待queue非空。

对pthreads condition的使用：  
1、变量类型pthread_cond_t。  
2、初始化，PTHREAD_COND_INITIALIZER或pthread_cond_init()。  
3、等待条件满足，pthread_cond_wait(condition, mutex)。这里显示了pthreads中condition和mutex的紧密关联天性，调用前mutex需被加锁，而在wait时，此函数会自动解锁mutex。而当条件满足时——即其他地方调用signal时，此函数会自动加锁mutex，之后程序员需自行负责解锁。  
4、唤醒。有两种方式，pthread_cond_signal()和pthread_cond_broadcast()，后者用于多个线程wait时。  
5、销毁pthread_cond_destroy()。  
6、attr相关操作。  

Condition封装pthreads condition，在ctor/dtor中做pthread_cond_init/destroy，提供wait()/notify()/notifyAll()，分别是对pthreads中对应函数的简单封装。Condition类的一个小细节是定义MutexLock&，而非定义MutexLock，因为后者构造一个新的变量，违背语义。

{% highlight cpp %}
class Condition : boost::noncopyable {
public:
    explicit Condition(MutexLock& mutex) : mutex_(mutex) {
        pthread_cond_init(&pcond_, NULL);
    }
    ~Condition() { pthread_cond_destroy(&pcond_); }

    void wait() {
        pthread_cond_wait(&pcond_, mutex_.getPthreadMutex());
    }
    void notify() {
        pthread_cond_signal(&pcond_);
    }
    void notifyAll() {
        pthread_cond_broadcast(&pcond_);
    }
private:
    MutexLock& mutex_;
    pthread_cond_t pcond_;
};
{% endhighlight %}

###Thread

pthreads中线程的操作：  
1、线程描述符pthread_t。  
2、创建线程pthread_create()。  
3、终止线程pthread_exit()。  
4、join和detach。  

Thread封装pthreads基本的线程管理，其中的数据成员有：  
1、线程描述符pthread_t  
2、线程pid_t id  
3、线程的工作函数，为函数指针类型，通过boost::function定义  
4、线程名字  
5、标志线程是否启动的bool started_  
6、AtomicInt32变量numCreated_，保存创建的线程数目  

线程是何时开始执行的？想起这个问题时，我楞了一下。线程是创建之后就“立即”执行了——“立即”意为你无需再显示调用pthread_start()之类的函数。所以Thread::start()函数仅仅是封装pthread_create()。列出Thread的函数：  
1、Thread::start()，封装pthread_create()  
2、Thread::join()，封装pthread_join()  
3、线程工作函数func_由boost::function定义，由用户传入，但func_并不直接传入pthread_create()，而是被startThread做了一层wrap，增加了try-catch保护  

所以，Thread是简单的，其亮点之一是boost::function定义线程工作函数，之二是存储了pid——Thread是如何自省得到pid的了？下一小节说明。

###namespace CurrentThread

CurrentThread是一个namespace，不是一个class。其主要目的是获取当前线程的名字和ID，可以想见是通过系统调用实现的，但非直接调用gettid()，而是调用syscall(SYS_gettid)，这么做应是为避免[glibc cache](http://xanpeng.github.com/linux/2012/05/19/ps-getpid.html)。

{% highlight cpp %}
pid_t gettid() {
	return static_cast<pid_t>(::syscall(SYS_gettid));
}
{% endhighlight %}

###ThreadPool

顾名思义，将线程放在集合中，形成线程池。好处是节省频繁创建销毁现场的开销、控制线程的数量。  
ThreadPool的主要功能：  
1、定义Thread队列，控制线程数目。  
2、定义任务队列deque<Task>，如果用户高频提交任务，线程池处理不过来，这个队列就会很长。  
3、deque<Task>被MutexLock+Condition保护。  

线程终止有[多种方式](https://computing.llnl.gov/tutorials/pthreads/#CreatingThreads)，其中之一便是线程顺利完成任务而“自然”退出。于是，这个有一个疑问：ThreadPool的线程必然不能完成一个任务就退出，它们要循环地处理用户提交的Task，对此ThreadPool的实现方法是简单巧妙的：

{% highlight cpp %}
while (running_) {
	Task task(take());
	if (task) {	
		task();
	}
}
{% endhighlight %}

然而，ThreadPool又如何正常退出？ThreadPool::stop()通过两个步骤退出：  
1、设置running_=false，使得线程不再从deque<Task>中获取新任务。  
2、等待(join)每一个线程结束。  

但还有一个细节：何时调用ThreadPool::stop()？不考虑时机，随意调用stop()，线程池终止时可能还有Task没有完成。  
ThreadPool并不提供策略处理此场景，而由用户在外层自行处理。ThreadPool_test.cc中采用的方案是CountDownLatch，即往deque<Task> push一个countdown任务，此任务完成之后，调用ThreadPool::stop()。用户只要在countdown之后不再push Task即可，便于控制。——这也说明，目前ThreadPool是按FIFO顺序处理任务的，未作其他策略。

下面给出代码，先列出用户代码，用户代码揭示ThreadPool的对外接口，是代码分析入口。为了节省篇幅，下面的代码调整了格式，删除了不紧要内容。从中可以看出boost::function+boost::bind用起来是多么简单优美！

{% highlight cpp %}
// ThreadPool_test.cc
void print() { 
	printf("tid=%d\n", muduo::CurrentThread::tid()); 
}
void printString(const std::string& str) { 
	printf("tid=%d, str=%s\n", muduo::CurrentThread::tid(), str.c_str()); 
}
int main() {
    muduo::ThreadPool pool("MainThreadPool");
    pool.start(5);	// 池中线程数目固定为5，ThreadPool没有提供调整池大小的方法。

    pool.run(print);
    pool.run(print);
    for (int i = 0; i < 100; ++i) {
        char buf[32];
        snprintf(buf, sizeof buf, "task %d", i);
        pool.run(boost::bind(printString, std::string(buf)));
    }

    muduo::CountDownLatch latch(1);
    pool.run(boost::bind(&muduo::CountDownLatch::countDown, &latch));
    latch.wait();
    pool.stop();
}
{% endhighlight %}

{% highlight cpp %}
// ThreadPool.h
class ThreadPool : boost::noncopyable {
public:
    typedef boost::function<void ()> Task;

    explicit ThreadPool(const string& name = string());
    ~ThreadPool();
    void start(int numThreads);	// 创建线程池，创建线程
    void stop();	// 关闭线程池
    void run(const Task& f);	// 放入任务

private:
    void runInThread();	// 内部方法，主体是while循环
    Task take();	// 获取一个Task

    MutexLock mutex_;
    Condition cond_;
    string name_;
    boost::ptr_vector<muduo::Thread> threads_;
    std::deque<Task> queue_;
    bool running_;
};
{% endhighlight %}

{% highlight cpp %}
// ThreadPool.cc
void ThreadPool::start(int numThreads) {
    assert(threads_.empty());
    running_ = true;
    threads_.reserve(numThreads);
    for (int i = 0; i < numThreads; ++i) {
        char id[32];
        snprintf(id, sizeof id, "%d", i);
        threads_.push_back(new muduo::Thread(boost::bind(&ThreadPool::runInThread, this), name_+id));
        threads_[i].start();
    }
}
void ThreadPool::stop() {
    running_ = false;
    cond_.notifyAll();
    for_each(threads_.begin(), threads_.end(), boost::bind(&muduo::Thread::join, _1)); // _1是Thread类型
}
void ThreadPool::run(const Task& task) {
    if (threads_.empty()) task();	// 这只是一个小trick，如果用户没有调用start()，就是此处的单线程模式。
    else {
        MutexLockGuard lock(mutex_);
        queue_.push_back(task);
        cond_.notify();	// 通知等待task queue非空的线程。
    }
}
void ThreadPool::runInThread() {	// 线程的主体函数
    try {
        while (running_) {
            Task task(take());
            if (task) task();
        }
    }
	...
}
ThreadPool::Task ThreadPool::take() {
    MutexLockGuard lock(mutex_);
    // always use a while-loop, due to spurious wakeup
    while (queue_.empty() && running_) {
        cond_.wait();
    }
    Task task;
    if(!queue_.empty()) {
        task = queue_.front();
        queue_.pop_front();
    }
    return task;
}
{% endhighlight %}

###ThreadLocal，BlockQueue，CountDownLatch

线程局部变量，平时几无接触，暂不分析。BlockQueue、CountDownLatch也只是两个辅助手段，非muduo.thread的主要功能，也暂不分析。