---
title: 理解boost::bind
layout: post
category: coding
tags: cpp boost library
---

*好久没写博，就像已没有了梦想一样（汗！），每天也觉不充实。实是因为最近难得忙，我不得不投入全部心力。今日有阶段性成果，于是晚间拿起放下已久的C++。*

我在看[muduo中的C++多线程库](http://blog.csdn.net/Solstice/article/details/5829421)，途中遇到boost::bind技法，陈硕在[“以boost::function和boost:bind取代虚函数”](http://blog.csdn.net/Solstice/article/details/3066268)中“高屋建瓴”地探讨其好处，为我现在不能理解。本文打算理解boost::bind，搜寻得到“[详细解析boost中bind的实现](http://blog.csdn.net/hengyunabc/article/details/7773250)”，很不错，本文的代码和理解多引自此文。

###STL bind1st的实现

为了区别STL实现，下面例子中都在名字后加上“2”。可以看出bind1st的原理，实际上是把参数保存起来，等到调用时，再取出使用。  
这份代码还是好理解的，你会发现后面的boost::bind代码，会让人崩溃！

{% highlight cpp %}
#include <iostream>
#include <algorithm>
using namespace std;

template<typename _Arg1, typename _Arg2, typename _Result>
struct binary_function2 {
	// 一个不包含数据的类，包含3个typedef，我理解其作用只是方便后面的抽象和使用。
	// 7.27 UPD: 定义了三个类型，binary_function2::first_argument_type等。
    typedef _Arg1 first_argument_type;
    typedef _Arg2 second_argument_type;
    typedef _Result result_type;
};

template<typename _Tp>
struct equal_to2 : public binary_function2<_Tp, _Tp, bool> {
    bool operator()(const _Tp& __x, const _Tp& __y) const {
        return __x == __y;
    }
};

template<typename _Arg, typename _Result>
struct unary_function2 {
    typedef _Arg argument_type;
    typedef _Result result_type;
};

template<typename _Operation>
class binder1st2 : public unary_function2<
    typename _Operation::second_argument_type, // typename指明second_xxx是_Operation中的一个类型，而非静态成员
    typename _Operation::result_type> {
protected:
    _Operation op;
    typename _Operation::first_argument_type value;

public:
    binder1st2(const _Operation& __x,
        const typename _Operation::first_argument_type& __y)
        : op(__x), value(__y) {}

    typename _Operation::result_type operator()
            (const typename _Operation::second_argument_type& __x) const {
        return op(value, __x);
    }

    typename _Operation::result_type operator()
            (typename _Operation::second_argument_type& __x) const {
        return op(value, __x);
    }
};

template<typename _Operation, typename _Tp>
inline binder1st2<_Operation> bind1st2(const _Operation& __fn, const _Tp& __x) {
    typedef typename _Operation::first_argument_type _Arg1_type;
    return binder1st2<_Operation>(__fn, _Arg1_type(__x));
}

int main()
{
    int numbers[] = { 10,20,30,40,50,10 };

    binder1st2<equal_to2<int> > equal_to_10(equal_to2<int>(), 10);
    int cx = std::count_if(numbers, numbers + 6, equal_to_10);
    cout << "There are " << cx << " elements that are equal to 10.\n";

	// cx = std::count_if(numbers, numbers + 6, bind1st(equal_to2<int>(), 10));
    cx = std::count_if(numbers, numbers + 6, bind1st2(equal_to2<int>(), 10));
    cout << "There are " << cx << " elements that are equal to 10.\n";
}
{% endhighlight %}

###boost::bind的实现

下面的代码是前述博文作者从boost::bind源码中“抠”出来的。先列出main.cc，再列bind.h，你会发现，这代码是妖！

{% highlight cpp %}
// main.cc
#include <iostream>
#include "bind.h"
using namespace std;

void tow_arguments(int i1, int i2) { cout << i1 << i2 << endl; }

class Test {};
void test_class(Test t, int i) { cout << "test_class(Test, int), " << i << endl; }

int main() {
    int i1 = 1, i2 = 2;
    (boost::bind2(&tow_arguments, 123, _1))(i1, i2);
    (boost::bind2(&tow_arguments, _1, _2))(i1, i2);
    (boost::bind2(&tow_arguments, _2, _1))(i1, i2);
    (boost::bind2(&tow_arguments, _1, _1))(i1, i2);

    (boost::bind2(&tow_arguments, 222, 666))(i1, i2);

    Test t;
    (boost::bind2(&test_class, _1, 666))(t, i2);
    (boost::bind2(&test_class, _1, 666))(t);
    (boost::bind2(&test_class, t, 666))();

    return 0;
}
{% endhighlight %}

{% highlight cpp %}
// bind.h
#ifndef BOOST_BIND_HPP
#define BOOST_BIND_HPP
#include <iostream>
using namespace std;

namespace boost
{

template<typename T> struct is_placeholder {
    enum _vt { value = 0 };
};

template<int I> struct arg {
    arg() {}
};

template<typename T> struct type {};

namespace _bi // implementation details
{

template<typename A1> struct storage1 {
    explicit storage1(A1 a1) : a1_(a1) { cout << "storage1(A1 a1)" << endl; }
    A1 a1_;
};

template<int I> struct storage1<boost::arg<I> > {
    explicit storage1(boost::arg<I>) { cout << "storage1(boost::arg<I>)" << endl; }
    static boost::arg<I> a1_() { return boost::arg<I>(); }
};

template<int I> struct storage1<boost::arg<I> (*)()> {
    explicit storage1(boost::arg<I> (*)()) { cout << "storage1(boost::arg<I> (*)())" << endl; }
    static boost::arg<I> a1_() { return boost::arg<I>(); }
};

template<typename A1, typename A2> struct storage2: public storage1<A1> {
    typedef storage1<A1> inherited;
    storage2(A1 a1, A2 a2) : storage1<A1>(a1), a2_(a2) { cout << "storage2(A1 a1, A2 a2)" << endl; }
    A2 a2_;
};

template<typename A1, int I> struct storage2<A1, boost::arg<I> > : public storage1<A1> {
    typedef storage1<A1> inherited;
    storage2(A1 a1, boost::arg<I>) : storage1<A1>(a1) { cout << "storage2(A1 a1, boost::arg<I>)" << endl; }
    static boost::arg<I> a2_() { return boost::arg<I>(); }
};

template<typename A1, int I> struct storage2<A1, boost::arg<I> (*)()> : public storage1< A1> {
    typedef storage1<A1> inherited;
    storage2(A1 a1, boost::arg<I> (*)()) : storage1<A1>(a1) { cout << "storage2(A1 a1, boost::arg<I> (*)())" << endl; }
    static boost::arg<I> a2_() { return boost::arg<I>(); }
};

// result_traits
template<typename R, typename F> struct result_traits {
    typedef R type;
};
struct unspecified {
};
template<typename F> struct result_traits<unspecified, F> {
    typedef typename F::result_type type;
};
// value
template<typename T> class value {
public:
    value(T const & t) : t_(t) {}
    T & get() { return t_; }
private:
    T t_;
};

// type
template<typename T> class type {};

// unwrap
template<typename F> struct unwrapper {
    static inline F & unwrap(F & f, long) { return f; }
};

// listN
class list0 {
public:
    list0() {}

    template<typename T> T & operator[](_bi::value<T> & v) const {
        cout << "list0 T & operator[](_bi::value<T> & v)" << endl;
        return v.get();
    }
    template<typename R, typename F, typename A> R operator()(type<R>, F & f, A &, long) {
        cout << "list0 R operator()(type<R>, F & f, A &, long)" << endl;
        return unwrapper<F>::unwrap(f, 0)();
    }
    template<typename F, typename A> void operator()(type<void>, F & f, A &, int) {
        cout << "list0 void operator()(type<void>, F & f, A &, int)" << endl;
        unwrapper<F>::unwrap(f, 0)();
    }
};

template<typename A1> class list1: private storage1<A1> {
private:
    typedef storage1<A1> base_type;
public:
    explicit list1(A1 a1) : base_type(a1) {}
    A1 operator[](boost::arg<1>) const { return base_type::a1_; }
    A1 operator[](boost::arg<1> (*)()) const { return base_type::a1_; }
    template<typename T> T & operator[](_bi::value<T> & v) const { return v.get(); }
    template<typename R, typename F, typename A> R operator()(type<R>, F & f, A & a, long) {
        return unwrapper<F>::unwrap(f, 0)(a[base_type::a1_]);
    }
};
template<typename A1, typename A2> class list2: private storage2<A1, A2> {
private:
    typedef storage2<A1, A2> base_type;
public:
    list2(A1 a1, A2 a2) : base_type(a1, a2) {}
    A1 operator[](boost::arg<1>) const {
        cout << "list2 A1 operator[](boost::arg<1>)" << endl;
        return base_type::a1_;
    }
    A2 operator[](boost::arg<2>) const {
        cout << "list2 A1 operator[](boost::arg<2>)" << endl;
        return base_type::a2_;
    }
    A1 operator[](boost::arg<1> (*)()) const {
        cout << "list2 A1 operator[](boost::arg<1> (*)())" << endl;
        return base_type::a1_;
    }
    A2 operator[](boost::arg<2> (*)()) const {
        cout << "list2  A1 operator[](boost::arg<2> (*)())" << endl;
        return base_type::a2_;
    }
    template<typename T> T & operator[](_bi::value<T> & v) const {
        cout << "T& operator[](_bi::value<T> & v)" << endl;
        return v.get();
    }
    template<typename R, typename F, typename A> R operator()(type<R>, F& f, A& a, long) {
        return f(a[base_type::a1_], a[base_type::a2_]);
    }
};

// bind_t
template<typename R, typename F, typename L> class bind_t {
public:
    typedef bind_t this_type;
    bind_t(F f, L const & l) : f_(f), l_(l) {}

    typedef typename result_traits<R, F>::type result_type;
    result_type operator()() {
        cout << "bind_t::result_type operator()()" << endl;
        list0 a;
        return l_(type<result_type>(), f_, a, 0);
    }
    template<typename A1> result_type operator()(A1 & a1) {
        list1<A1 &> a(a1);
        return l_(type<result_type>(), f_, a, 0);
    }
    template<typename A1, typename A2> result_type operator()(A1 & a1, A2 & a2) {
        list2<A1 &, A2 &> a(a1, a2);
        return l_(type<result_type>(), f_, a, 0);
    }
private:
    F f_;
    L l_;
};

template<typename T, int I> struct add_value_2 {
    typedef boost::arg<I> type;
};
template<typename T> struct add_value_2<T, 0> {
    typedef _bi::value<T> type;
};
template<typename T> struct add_value {
    typedef typename add_value_2<T, boost::is_placeholder<T>::value>::type type;
};
template<typename T> struct add_value<value<T> > {
    typedef _bi::value<T> type;
};
template<int I> struct add_value<arg<I> > {
    typedef boost::arg<I> type;
};
template<typename R, typename F, typename L> struct add_value<bind_t<R, F, L> > {
    typedef bind_t<R, F, L> type;
};
// list_av_N
template<typename A1> struct list_av_1 {
    typedef typename add_value<A1>::type B1;
    typedef list1<B1> type;
};
template<typename A1, typename A2> struct list_av_2 {
    typedef typename add_value<A1>::type B1;
    typedef typename add_value<A2>::type B2;
    typedef list2<B1, B2> type;
};
} // namespace _bi

// function pointers
template<typename R> _bi::bind_t<R, R (*)(), _bi::list0> bind2(R (*f)()) {
    typedef R (*F)();
    typedef _bi::list0 list_type;
    return _bi::bind_t<R, F, list_type>(f, list_type());
}
template<typename R, typename B1, typename A1>
_bi::bind_t<R, R (*)(B1), typename _bi::list_av_1<A1>::type> bind2(R (*f)(B1), A1 a1) {
    typedef R (*F)(B1);
    typedef typename _bi::list_av_1<A1>::type list_type;
    return _bi::bind_t<R, F, list_type>(f, list_type(a1));
}
template<typename R, typename B1, typename B2, typename A1, typename A2>
_bi::bind_t<R, R (*)(B1, B2), typename _bi::list_av_2<A1, A2>::type> bind2(R (*f)(B1, B2), A1 a1, A2 a2) {
    typedef R (*F)(B1, B2);
    typedef typename _bi::list_av_2<A1, A2>::type list_type;
    return _bi::bind_t<R, F, list_type>(f, list_type(a1, a2));
}

} // namespace boost

namespace {
boost::arg<1> _1;
boost::arg<2> _2;
}
#endif
{% endhighlight %}

理解这段代码，实非易事，如此多的模板使用让人眼花缭乱。从示例代码可以看出，bind记录参数，在调用函数时能灵活地传入参数。但实际上示例代码没有展示bind的真正效力。此处也不追问，且看这段妖码。分功能和语法技巧来分析这段代码。

功能：通过cout输出跟踪调用路径，能理解bind.h中错综复杂抽象的作用，下面仅截取第一个调用的输出：

{% highlight text %}
bind2(R (*f)(B1, B2), A1 a1, A2 a2)
storage1(A1 a1)
storage2(A1 a1, boost::arg<I>)
bind_t(F f, L const & l)
bind_t.operator()(A1 & a1, A2 & a2)
storage1(A1 a1)
storage2(A1 a1, A2 a2)
list2 A1 operator[](boost::arg<1> (*)())
T& operator[](_bi::value<T> & v)
1231
{% endhighlight %}

可看出，bind.h仅提供了最多两个参数的bind。首个调用的执行流程：  
1、bind2先构造bind_t对象。  
1.1、先通过list2记录666和\_1。  
2、再调用bind_t的operator()。  
2.1、先通过list2记录i1和i2。  
2.2、返回list2 operator()。  
3、执行list2的operator()，即执行f(a[base_type::a1\_], a[base_type::a2\_])。  

于是此时对于storage1/storage2，list1/list2等有了初步理解，至于其他更多细节，以及细节后面体现的设计理念，非目前力所能及之事，留待升级后解决。

而对于语法技巧，我有很多疑问需要解决：  
1、private继承的含义。  
2、list2的operator()(type<R>, F& f, A& a, long)定义，为什么long没有变量名？为什么可以l\_(type<result_type>(), f\_, a, 0)这么调用？  
3、boost::arg/type这样的空类的作用是什么？  
4、构造函数explicit storage1(boost::arg\<I\>)的参数也只是类型，没有变量名，为什么？而且提供static boost::arg\<I\> a1\_() { return boost::arg\<I\>(); }的含义是什么？  
5、unwrapper的作用是什么？  
6、很多空类仅包含typedef，含义是什么？  

不打算立即全力解决这些问题，而是继续学习muduo.thread，这些问题随知识积累将逐步解决。  
可以看出，C++模板使用变化万端，整个代码难见C的直白风格，C++程序会有类型体系，变量、函数等都是类型，把这个观点放在心中，十分利于理解代码！

