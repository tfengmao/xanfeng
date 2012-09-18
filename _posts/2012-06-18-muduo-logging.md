---
title: muduo's c++ high-perf logging
layout: post
tags: muduo c++ logging performance nontype template
category: programming
---

*接下来我会围绕muduo logging持续更新这一篇博文, 主要涉及c++技法和高性能日志库需要注意的问题.*

---

#编译muduo

首先当然要编译, 当然最好依据"官方"文档了: [MuduoManual.pdf](http://www.chenshuo.com/pdf/MuduoManual.pdf). 过程无需细说, 值得注意两点:  
- 代码依平台不同, 可能略有小bug, 阻止编译通过, 很好解决.  
- 默认依赖的工具中有 doxygen, doxygen可能依赖graphviz, 直接编译可能报错, 可以看到脚本里有"cmake --graphviz=file ...", 偷懒的话可直接去除使用doxygen和graphviz.  

---

#C++部分

*本部分包含代码风格,经验,语法等内容, 我将它们混杂地放在一起.*

**boost**  
你会发现, muduo logging使用了很多boost的东西, 关于boost[这里](http://zh.highscore.de/cpp/boost/)有一个不错的资料, 是<The Boost C++ Libraries>的中译版.  
boost相关的话题, 由于内容很多, 将新开"[muduo使用的boost](http://xanpeng.github.com/2012/06/22/muduo-boost/)"来讨论.

**编码风格**  
我去年曾了解过[Google C++编码风格](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml), muduo logging的代码用的应该就是这个编码风格, 最突出的特点是内部变量全小写, 并以"_"结尾, 常量以"k"打头, 其余采用驼峰方式.  
初看来, muduo logging的代码看起来赏心悦目, 少量精简必要的注释点缀其间, 其他则由代码"自注释"了.  
整个代码组织和观感非常符合我的审美:)

**internal typedefs**  
[Internal typedefs in C++ - good style or bad style?](http://stackoverflow.com/questions/759512/internal-typedefs-in-c-good-style-or-bad-style).  
参考muduo::LogStream的代码, 我觉得这样用很直观.

    class LogStream : boost::noncopyable
    {
        typedef LogStream self;
    public:  
        ...
        self& operator<<(short);
        self& operator<<(unsigned short);
        self& operator<<(int);
        self& operator<<(unsigned int);
        self& operator<<(long);
        ...

**用嵌套namespace区分语义和细节**  
以LogStream为例, LogStream位于namespace muduo中, 它内部使用了buffer, 用的是detail::FixedBuffer, FixedBuffer属于内部细节, 被放置到namespace muduo::detail中.

    class LogStream : boost::noncopyable
    {
        typedef LogStream self;
     public:
        typedef detail::FixedBuffer<detail::kSmallBuffer> Buffer;
        ...

**非型别类模板(Nontype Class Template)**  
从这个"拗口"的名字你大致可以猜测这是TW同胞的说法了. 不错, 我看的是侯捷大牛等人翻译的<C++ Templates全览>.  
muduo logging使用此物的一个地方是FixedBuffer:

    template<int SIZE>
    class FixedBuffer : boost::noncopyable
    {
        ...
    private:
        char data_[SIZE];
        ...
    }

这么做的好处便是, 你可以FixedBuffer<1024>, 也可以FixedBuffer<2048>, 很灵活. 据我去年看书的记忆, FixedBuffer<1024>和FixedBuffer<2048>是不同的两个类型, 是FixedBuffer类型族的两员(*我yy的...*).  
*记得我看书时, 曾看到为什么有模板, 模板的根本局限是什么, 这些问题就留待以后解决吧, 现在是打怪升级刷经验期间, 这些"道"的东西不能现在强悟.*

**函数模板(Function Template)**  
在类中使用了函数模板, 和类是否是模板没有关系. 同样, 貌似是有"函数族"的说法的.

    template<typename T>
    void LogStream::formatInteger(T v)
    {
      if (buffer_.avail() >= kMaxNumericSize)
      {
        size_t len = convert(buffer_.current(), v); 
        buffer_.add(len);
      }
    }

**使用boost::noncopyable**  
在前面的示例代码中可看到noncopyable的踪迹, 这么做的目的是防止用户错误地使用copy-ctor和赋值(一般是错误地使用了编译器提供的copy-ctor, ctor等), 从设计上做严格的限制.  
更多资料:  
- [More C++ Idioms/Non-copyable Mixin](http://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Non-copyable_Mixin).  
- Effective C++ 3rd, 条款06:若不想使用编译器自动生成的函数, 就该明确拒绝; 条款14:在资源管理类中小心copying行为.

**implicit_cast**  
c++有四种类型转换符: static_cast, const_cast, dynamic_cast, reinterpret_cast. 据说implicit_cast本有希望成为第五种的, 但由于提交得太晚, 所以被错过.  

    // The From type can be inferred, so the preferred syntax for using
    // implicit_cast is the same as for static_cast etc.:
    //
    //   implicit_cast<ToType>(expr)
    //
    // implicit_cast would have been part of the C++ standard library,
    // but the proposal was submitted too late.  It will probably make
    // its way into the language in the future.
    template<typename To, typename From>
    inline To implicit_cast(From const &f) {
      return f;
    }

使用时, 可以这么用: 

    if (implicit_cast<size_t>(avial()) > len)

首先这种用法使用了function template argument deduction机制(参考"C++ Templates全览" 2.3), 只指出返回值类型, 参数类型交给编译器去推导.  
其次, implicit_cast的语义是什么? 总的来说, implicit_cast是一个safer, 限制更多的"操作符".  
鉴于c++中类型转换的类型蛮多+我脑残失忆竟不能理解static_cast为何物, 决定新开[一篇post](http://xanpeng.github.com/2012/06/19/cpp-type-conversion/)单讲type conversion, 包含了implicit_cast.

**template ctor和explicit instantiations**  
"[C++ template constructor](http://stackoverflow.com/questions/3960849/c-template-constructor)"的accepted answer说, 不方便调用template ctor.  

    // 定义 @LogStream.h
    class Fmt // : boost::noncopyable
    {
    public:
        template<typename T>
        Fmt(const char* fmt, T val);
        ...
    };

    // 实现 @LogStream.cc
    template<typename T>
    Fmt::Fmt(const char* fmt, T val)
    {
        BOOST_STATIC_ASSERT(boost::is_arithmetic<T>::value == true);
        length_ = snprintf(buf_, sizeof buf_, fmt, val);
        assert(static_cast<size_t>(length_) < sizeof buf_);
    }

    // 显示实例化, 可参考<C++ template全览> 6.2节
    // Explicit instantiations
    template Fmt::Fmt(const char* fmt, char);
    template Fmt::Fmt(const char* fmt, short);
    template Fmt::Fmt(const char* fmt, unsigned short);
    template Fmt::Fmt(const char* fmt, int);
    template Fmt::Fmt(const char* fmt, unsigned int);
    template Fmt::Fmt(const char* fmt, long);
    template Fmt::Fmt(const char* fmt, unsigned long);
    template Fmt::Fmt(const char* fmt, long long);
    template Fmt::Fmt(const char* fmt, unsigned long long);
    template Fmt::Fmt(const char* fmt, float);
    template Fmt::Fmt(const char* fmt, double);

**operator as non-member function**  
wikibooks "[Operator overloading](http://en.wikibooks.org/wiki/C++_Programming/Operators/Operator_Overloading)"列出了操作符重载所有层面的技能点, 如不可重载的操作符, [不]作为member function重载等.  
当operator定义为member func时, 需显式传递的参数可以减少一个, 因为类中都隐含一个this对象, 因此二元操作符只需要指定一个参数, 一元操作符不需要指定参数.  
当operator定义为non-member func时, 需显示传递的第一个参数就是"this"对象.

    // as member func
    Vector2D Vector2D::operator+(const Vector2D& right) const {...}
    // as non-member func
    Vector2D operator+(const Vector2D& left, const Vector2D& right) {...}

**为什么**要实现为non-member func? 怎么调用?

    // muduo logging中这么使用, 其目的是, LogStream和Fmt是相互独立的两个类, 
    // 如果重载为member func, 需要"LogStream& operator<<(const Fmt&)"这么写, 
    // 这样增加了LogStream和Fmt的耦合度, 所以实现为non-member func.
    inline LogStream& operator<<(LogStream& s, const Fmt& fmt)
    {
        s.append(fmt.data(), fmt.length());
        return s;
    }

    // 调用方式当然是一致的, 要不然怎么算重载呢.
    LogStream os;
    os << Fmt("%4.2f", 1.2); // 调用 non-member func
    os << "hello world"; // 调用 member func

**#pragma GCC diagnostic ignored "-Wold-style-cast"**  
[\#pragma](http://gcc.gnu.org/onlinedocs/cpp/Pragmas.html)是用来向compiler提供额外信息的一种方式. 而"[#pragma GCC diagnostic kind option](http://gcc.gnu.org/onlinedocs/gcc/Diagnostic-Pragmas.html#Diagnostic-Pragmas)"是gcc提供的机制, 可让用户选择enable/disable某些diagnostics.  
而Wold-style-cast是用以提示c++代码中使用old-style(C-style) cast的, 可以在"[GCC Command Options](http://gcc.gnu.org/onlinedocs/gcc-3.0.4/gcc_3.html)"中看到更多gcc选项.  
另外提一个"[#pragma once](http://en.wikipedia.org/wiki/Pragma_once)", 它的作用类似于[include guard](http://en.wikipedia.org/wiki/Include_guard).

---

#logging部分

*本部分设计的c++技法都放在前面的"c++部分"讲述, 这里仅列出特别的实现策略.*

[MuduoManual.pdf](http://www.chenshuo.com/pdf/MuduoManual.pdf)主要是muduo的文档, 而muduo是一个高性能的网络lib, 可能是我接下来的学习目标. 在编译muduo的时候, 根据该manual的链接看到陈硕的博文"[学之者生，用之者死——ACE历史与简评](http://blog.csdn.net/Solstice/article/details/5364096)", giantchen行文谨慎, 资料翔实, 论述有据, 认知深刻, 非常推荐. 以后对[ACE](http://www.cs.wustl.edu/~schmidt/ACE.html)要谨慎, 同时强烈阻止任何使用它的企图.

**FixedBuffer**  
这个类最大的疑问是, 为什么有cookie? 什么意思? 有什么用?  
FixedBuffer的设计是简单的: 设置有一个定长数组, 一个游标指针, 一些辅助方法.  
当然有一些不平常的地方:  
- 使用bzero往buffer中写零, 而不是用memset. 搜索得: [对于小数组, bzero快过memset, memset甚至慢过for(...)](http://hi.baidu.com/wg_wang/blog/item/ee41553125c9db1feac4af41.html "bzero & memset置零的性能比较"), 其原因是[memset()是逐字节处理的](http://fdiv.net/2009/01/14/memset-vs-bzero-ultimate-showdown "memset() vs. bzero()-Ultimate Showdown")?  
- 提供的append()方法使用memcpy(), 将数据从from拷贝到buffer中.  
- 还提供了一个add()方法, 这个方法只是更新了buffer存放量计数而已, 但在调用add()之前, 用户必须已经往buffer中写入数据, 采用直接操作buffer的方式, 避免了内存拷贝.

**LogStream**  
初看起来LogStream很普通, 它使用了FixedBuffer, 设置buffer大小为4000字节.  
不平凡之处:  
- typedef LogStream self.  
- 涉及了sprintf和Grisu3 stringify double的问题, 为我原本不知. 搜索得giantchen的"[C++ 工程实践(7)：iostream 的用途与局限](http://www.cnblogs.com/Solstice/archive/2011/07/17/2108715.html)", 更是宝藏!  

**LogStream_bench.cc**  
这个函数提供了sprintf, ostringstream和LogStream的性能对比, 称之为bench. *原来bench可以是这样简单?*  
- 对int, double, int64_t, void*四种类型数据分别做了1000000次操作, 在64位机器上LogStream的性能一般. *不过采用这种bench方案是否合适?*  

**LogFile**  
LogFile代表日志文件, 包含日志文件的操作方法.  
- 底层当然还是由FILE支撑的, 没有什么特别.  
- 提供了rollFile()机制, 由写入日志的条数触发.  
- 提供定期flush机制, 提高效率.  
- rollfile和flush的控制参数都可以通过ctor设置.  
- 为多线程提供了锁定机制, 使用MutexLock, 包装的是pthread_mutex_t.  
- 最终写入文件时调用的是[non-locking stdio functions](http://www.kernel.org/doc/man-pages/online/pages/man3/unlocked_stdio.3.html)的fwrite_unlocked(), 应是因为加锁交由LogFile层, stdio层无需再加锁, 以节省时间.  
- LogFile并非单独使用的, 而是被Logger调用的, Logger才是面向用户的界面.  

**Logger**  
顾名思义, 这个类提供用户接口, 实现多线程同步日志功能.  
- 如一般的, 定义了日志级别.  
- 使用LogStream供消息输入, 可知LogStream的buffer是有限的, 所以单行长度是受限的.  
- 使用LogFile将内容写入到文件.  
- 定义LOG_INFO等宏, 如LOG_INFO定义为muduo::Logger(__FILE__, __LINE__).stream().  
- 所以使用方法为: LOG_INFO << "hello".  
- 通过全局的OutputFunc, FlushFunc函数指针关联LogStream和LogFile, 在Logger的dtor中调用OutputFunc, 如果级别为FATAL, 在dtor中立即调用FlushFunc.  
- OutputFunc, FlushFunc由使用者设置, 默认写到stdout.  
- **特性**: 由于LOG_INFO等定义为宏, 如下面示例代码所示, 如果日志级别不符合, 不需要打印日志, 此时不会带来任何开销.  

    #define LOG_INFO if (muduo::Logger::logLevel() <= muduo::Logger::INFO) \
          muduo::Logger(__FILE__, __LINE__).stream()
    // 使用
    LOGINFO << msg;
    // 相当于
    if (muduo::Logger::logLevel() <= muduo::Logger::INFO)
          muduo::Logger(__FILE__, __LINE__).stream() << msg;

Logger的亮点主要在实现代码上, 代码写的漂亮, 类关系抽象得很好.

**MutexLockGuard**  
此类的成员MutexLock封装pthread_mutex_t, 这是根本, 毋庸置疑.  
这个类采用的是某种锁设计模式吧, 在ctor中加锁, 在dtor中解锁, 使用时只需要创建MutexLockGuard变量即可.  
这么做的确方便, 而且由"dtor总被调用"来保证unlock, 避免编码遗漏引起的锁异常. 但是语义上把加解锁的动作隐藏起来, 感觉不习惯.

**AsyncLogging**  
所谓异步日志, 是指并不是在执行LOG_INFO等时就将msg写入文件, 因为写文件耗时长, 会影响主流程的效率.  
*在继续写文前, 再次指出, muduo logging的机制和我曾写过的c asynclogger大同小异, 都是采用buffer和后台线程. 但muduo logging优于asynclogger的地方是: 优美的实现代码, giantchen高屋建瓴的总结.*  
所以, AsyncLogging的做法是: frontend(LOG_INFO等)将消息写到buffer0, buffer0写满时使用buffer1, 并触发后台线程将buffer0的数据写到文件中.  
这些细节都在append()函数中, 当然为了保证时序, 其中是采用MutexLockGuard保护的.  
AsyncLogging和Logger是类似的, 只不过Logger的OutputFunc是直接写文件, 而AsyncLogging的OutputFunc是写buffer0/1的. 其余类似, 不再赘述.

**muduo logging总结**  
logging部分了解到此, 已无秘密, 剩下的是所使用的c++技法, 将继续更新.  
总结logging, 可以用作者的slide: "[Efficient logging in multithreaded c++ server](http://www.slideshare.net/chenshuo/efficient-logging-in-multithreaded-c-server/)". 从中可以看到全面的思路, 强烈推荐! 下面摘取重点:  
- diagnostic log.  
- 去除不必要的, 次要的功能, 只留下最基本也最重要的功能.  
- frontend(LOG_INFO等)要求易用, 低延时. 使用LogStream, 其中使用stack上分配的buffer.  
- 只支持写到本地文件, 通过LogFile实现.  
- log one file for one process.  
- 避免global locking和blocking writing, 采用双buffer, 避免写入buffer和"buffer->disk file"的冲突, 这一点比我的好!  
- 后台线程负责将日志写入文件. 在其中一个buffer满时触发写的动作.

