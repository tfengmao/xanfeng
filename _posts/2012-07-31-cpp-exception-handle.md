---
title: 理解C++异常处理
layout: post
category: programming
tags: cpp exception handle
---

在“[如何实现异常处理](http://xanpeng.github.com/programming/2012/07/12/exception-handle.html)”中，我希望了解一种编程语言是如何实现异常(exception)机制的，但最终没有达到预期，现在仍然纠结。但今天想到：我应是将异常处理想得太复杂，在我的旧印象里，如果使用了异常处理，程序将永远稳健地执行，即使遇到除0。这是痴心妄想，除0是需要做特殊处理的，否则terminate。

说到底，我并没有认清C++异常的定义。大多书籍没有专门讲述（除非被我错过），在stackoverflow上看到相关讨论，总结于此。  

C++眼里的异常，是software-detected errors，比如文件不存在这样的“异常”场景。程序员完全可以不用异常，而使用error code，就像在C语言中处理类似“异常”场景一样————所以，异常只是一种编程手段而已。  
除0，程序在cli输出“Floating point exception”，这里虽有“exception”字样，但FPE**不是**C++异常处理机制中的“Exception”，是**不可捕获**的。FPE是一种典型的hardware exception，发生FPE时操作系统的典型做法是向进程发送**SIGFPE**信号，如果没有特别处理此信号，进程的默认行为一般是terminate。

因此，异常可划分为“软件异常”和“system-level exception”，而大家一般讨论的是第一种，本文则专门讨论C++异常处理，不考虑FPE这样的system-level exception。

下列文章针对FPE和Exception的区别：  
1、[Why does GCC report a Floating Point Exception when I execute 1/0?](http://stackoverflow.com/questions/9366215/why-does-gcc-report-a-floating-point-exception-when-i-execute-1-0)  
2、[Dealing with Floating Point exceptions](http://stackoverflow.com/questions/2219244/dealing-with-floating-point-exceptions)  
3、[C++ : Catch a divide by zero error](http://stackoverflow.com/questions/4747934/c-catch-a-divide-by-zero-error)  
4、[How do I catch system-level exceptions in Linux C++?](http://stackoverflow.com/questions/618215/how-do-i-catch-system-level-exceptions-in-linux-c)  

---

产生本文的原因之一，便是解决了FPE和Exception的纠结，此时发现“[如何实现异常处理](http://xanpeng.github.com/programming/2012/07/12/exception-handle.html)”中的描述是不恰当的，我会在原文首部添加警告。  
产生本文的原因之二，便是同事问我，如果在构造函数中发生异常，会怎么办？对于C++异常处理（再次表明：不考虑FPE等），我阅读颇多，不过脑中一团浆糊。于是趁此机会做一整理和了解，并告诉他best practice!

将讨论下面的话题，所有讨论都限定于C++的异常处理（Exception Handling）：  
1、什么是Exception？  
2、编译器如何实现Exception Handling？（原来有此执念，是误以为要handle FPE，现已澄清，其实已不太有此念了）  
3、用还是不用Exception？  
4、如何用Exception？  
5、Exception-safe

###什么是Exception

C++的异常处理不包含FPE等system-level exception。所以，这里说的Exception只是一种类似于error code的处理方式。

C++标准异常的基类是class Exception，[stdexcept](http://www.cplusplus.com/reference/std/stdexcept/)中定义了一些常用的异常，分为两类：logic_error（domain_error，invalid_argument，length_error，out_of_range）和runtime_error（range_error，overflow_error，underflow_error），new会抛出bad_alloc异常。

###编译器如何实现Exception Handling

原来误认为exception handling也要处理FPE的时候，觉得很是神奇，不知编译器是如何和内核协调处理FPE的。但现在既已明白，不管FPE，那么编译器实现exception handling的方式也可以想见，不过是在用户态做策略而已。

略加搜索，找到一些G++实现异常处理的资料，且列于此：  
1、[Internal Architecture of the Compiler](http://www.math.utah.edu/docs/info/gxxint_1.html)  
2、[How is the C++ exception handling runtime implemented?](http://stackoverflow.com/questions/490773/how-is-the-c-exception-handling-runtime-implemented)  
3、[How a C++ compiler implements exception handling](http://www.codeproject.com/Articles/2126/How-a-C-compiler-implements-exception-handling)  

现在自无必要去了解这些实现策略，但关乎exception使用的细节，自然是要知道的，下面列出c++2003std “Exception handling”中的有趣内容：  
1、不能用goto，switch跳入try block或handler，这样打乱了try-catch的逻辑。  
2、抛出异常时，会有一个object被传递给handler，object类型用来确定调用哪一个handler。  
3、throw表达式创建的对象称为exception object，the memory is allocated in an unspecified way，其生命周期由handler确定，只要有handler在处理，exception object所占内存就仍保持。  
4、...

###用不用Exception Handling

经过查询资料，我的结论是：要用Exception Handling。

不用Exception的理由：  
1、有性能损耗吧？  
2、Google就不用Exception（[Google cppguide#Exception](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml#Exceptions)）。  

####使用Exception会带来性能损耗吗？

如果没有抛出异常，则没有性能损耗；如果抛出异常，则有部分性能损耗，不过此时如果使用error code，也仍然是有损耗的。  
——参考“[How do exceptions work(behind the scenes) in c++](http://stackoverflow.com/questions/307610/how-do-exceptions-work-behind-the-scenes-in-c)”。

而且，现在的我绝对不到考虑这么一点性能损耗的时候，首先现代编译器的性能值得相信，其次采用更好的算法才是提升性能的正途！

####Google为什么不用Exception

Google在他的cppguide上列出了使用exception的优劣，指出缺点是产生更大的binary file，（可能）要更多的compile time，不确切的程序执行路径，增加维护和调试难度，以及不当使用带来的风险（这个要用，自然就要用好了，汗！）。实际上，可以看出，Google最终决定不用exception，多是因为历史原因，他有很多历史的C++项目没有使用Exception。

另外“[We do not use C++ exceptions — What's the alternative? Let it crash?](http://stackoverflow.com/questions/3490106/we-do-not-use-c-exceptions-whats-the-alternative-let-it-crash)”和“[Are there any real-world cases for C++ without exceptions?](http://programmers.stackexchange.com/questions/113479/are-there-any-real-world-cases-for-c-without-exceptions)”都讨论了是否使用Exception。

延伸这个讨论内容，就到了剪裁使用C++，孟岩在“[编程语言的层次观点——兼谈C++的剪裁方案](http://blog.csdn.net/myan/article/details/1920)”就讨论了这个话题。C++语言特性很多，很多我还不能深刻理解，于是剪裁话题暂列于此，不作展开。

###如何使用Exception

考虑三个问题：  
1、普通情况下，使用异常的best practice。  
2、构造函数中是否能抛出异常，怎么处理？  
3、析构函数中是否能抛出异常，怎么处理？  

讨论这三个问题的资料：[wikibook#C++_Exception_Handling](http://en.wikibooks.org/wiki/C%2B%2B_Programming/Exception_Handling)，《More Effective C++》的条款10“在构造函数中防止资源泄漏”、条款11“禁止异常信息传递到析构函数外”。

####非ctor/dtor的Exception Handling

如何抛出异常？自然是通过throw表达式。前文c++2003std部分提到，throw创建一个临时的exception object，在handler生命周期内都有效。而且，标准并不规定在内存的何处（堆、栈、全局区、其他？）创建exception object(原文是allocated in an unspecified way)，于是也不知道exception object是如何被销毁的。  
所以，抛出异常的正确写法是：thrown by value，caught by (usually const) reference。

{% highlight cpp %}
void foo() {
	throw MyApplicationException();	// throw by value.
}
void bar() {
	try {
		foo();
	} catch (MyApplicationException const& e) { // caught by reference
		// Handle exception
	}
}
{% endhighlight %}

异常在没被handle之前，会循着调用路径往回传递，过程中发生“[stack unwinding](http://en.wikibooks.org/wiki/C%2B%2B_Programming/Exception_Handling#Stack_unwinding)”，栈会被清除，栈内局部变量生命触底，dtor被调用，局部变量内存被回收，不会发生内存泄漏，但在堆上分配的内存，除非特别处理（使用smart pointer），否则仍然会泄漏。看示例代码：

{% highlight cpp %}
#include <iostream>
#include <string>
using namespace std;

struct Foo {
    Foo(string name) : name_(name) { cout << "Foo('"<< name << "')" << endl; }
    ~Foo() { cout << "~Foo('" << name_ << "')" << endl; }
    string name_;
};

void f3() {
    Foo f("f3");
    throw "exception in f3()";
}

void f2() {
    Foo f("f2");
	Foo* nf = new Foo("f2.new");
    f3();
	delete nf;
}

void f1() {
    Foo f("f1");
    try {
        f2();
    } catch (const char* e) {
        cout << "exception: " << e << endl;
    }
}

int main() {
    f1();
    cout << "after call" << endl;
    return 0;
}
{% endhighlight %}

{% highlight text %}
# ./exception_test
Foo('f1')
Foo('f2')
Foo('f2.new')
Foo('f3')
~Foo('f3')
~Foo('f2')
exception: exception in f3()
~Foo('f1')
after call

如果不使用try-catch，程序直接abort，没发生stack unwinding，不过进程退出也释放了内存
# ./exception_test
Foo('f1')
Foo('f2')
Foo('f2.new')
Foo('f3')
terminate called after throwing an instance of 'char const*'
Aborted
{% endhighlight %}

####ctor中的Exception Handling

ctor出错时，抛出此异常是最好的做法（但也可以把可能出错的代码放入init()，保证ctor不会异常）。[How can I handle a constructor that fails?](http://www.parashift.com/c++-faq-lite/ctors-can-throw.html)  

> Throw an exception.  
> Constructors don't have a return type, so it's not possible to use return codes. The best way to signal constructor failure is therefore to throw an exception. If you don't have the option of using exceptions, the "least bad" work-around is to put the object into a "zombie" state by setting an internal status bit so the object acts sort of like it's dead even though it is technically still alive.

如果在ctor中抛出异常，会不会发生内存泄漏？  
ctor异常时，对象未完整构建，dtor不会被调用，但异常发生前的对象会被合理析构。这其实和上面f1()->f2->f3()的例子是同一问题，stack unwind保证dtor的调用。同样，在堆上分配的内存需要特别处理。  
ctor抛出异常前，可能创建了基本类型(int,long等)的变量、指针变量、用户自定义类型变量，C++都“平等地”看待这些不同的类型——即C++不会像Java那样自作主张把所有class变量都放在堆上。  
这里面指针变量比较特殊，ctor异常后，指针变量本身遵循同样的规则，而被正确清理，但是它指向的对象不能被自动清理，从而发生内存泄漏。同样地，可以使用smart pointer解决这类问题（exception-safe部分会细说）。  
看ctor异常示例代码：

{% highlight cpp %}
#include <iostream>
using namespace std;

struct B {
    B() { cout << "B()" << endl; }
    ~B()    { cout << "~B()" << endl; }
};

struct C {
    C() { cout << "C()" << endl; }
    ~C()    { cout << "~C()" << endl; }
};

struct D {
    D() { cout << "D()" << endl; }
    ~D()    { cout << "~D()" << endl; }
};

struct E {
    E() { cout << "E()" << endl; }
    ~E()    { cout << "~E()" << endl; }
};

struct A : public B, public C{
    A() : B(), C(), d(), e(new E) {	// 注意这是new
        throw "A().exception";
    }
    D d;
    E* e;		// 指针本身不泄漏，指向的e泄漏了！
};

int main()
{
    try {
        A a;
    } catch (const char* e) {
        cout << e << endl;
    }
    cout << "after call" << endl;
    return 0;
}
{% endhighlight %}

{% highlight text %}
# ./ctor_ex
B()
C()
D()
E()
~D()
~C()
~B()
A().exception
after call
{% endhighlight %}

####dtor中的Exception Handling

dtor不能抛出异常！  
如果dtor抛出异常，程序会立即终止。这是因为，如果在处理其他异常时，在stack unwind过程中执行dtor，dtor又抛出新的异常，此时C++ runtime system如何处理是好？！忽略新异常，不行！忽略旧异常，然后处理新异常，也不行！  
所以，C++语法规定，dtor抛出异常时，程序终止并杀死进程。

*上述内容参考C++ FAQ “[[17.9] How can I handle a destructor that fails?](http://www.parashift.com/c++-faq-lite/dtors-shouldnt-throw.html)”。*

###exception-safe

如何做异常安全（exception-safe）编程？*我在很久以前的“[C++ 异常安全](http://xanpeng.github.com/programming/2012/02/12/cpp-exception-safe.html)”中提及此问题，特此说明。*

什么是异常安全？

暂无时间来解答此问题。回答此问题需要经验积累，我想留待以后。  
不过，异常安全的重要一点是保证没有资源泄漏。防止资源泄漏，一般的做法便是smart pointer。

列出一些资料：  
1、[Exception-Safe Class Design, Part 1: Copy Assignment ](http://www.gotw.ca/gotw/059.htm)  
2、[More C++ Idioms/Copy-and-swap](http://en.wikibooks.org/wiki/More_C++_Idioms/Copy-and-swap)  
3、[C++: do you (really) write exception safe code?](http://stackoverflow.com/questions/1853243/c-do-you-really-write-exception-safe-code)  

---

列出我未发布的一篇旧文。

> 我们都知道，数据库有事务的概念。据我理解，数据库的“事务性”保证属于同一个transaction的操作要么都成功并进入新的状态，要么transaction失败并保持原有状态，绝对不允许出现的是“中间状态”，即transaction中的某些操作成功，某些操作失败，这是不被允许的情景。  
> 我们在C++编程中也有同样的考虑：一个函数的执行是成功，还是失败，如果失败，是否会“部分成功”，即是否会产生副作用。本文将考虑这个问题，借用数据库事务的概念，我们称该问题为C++“事务一致性”问题。  
> 我们希望有一个工具包帮助解决问题，如果根本不存在这样的工具包，我们希望得到一种“指导思想”帮助我们认清并尽量解决C++“事务一致性”的问题。
> 
> 实际上我们可以将“事务一致性"问题转化为异常安全性(exception-safe)问题。如果函数执行过程中不会发生任何异常，则函数将执行成功，我们说该函数的“事务”成功完成。  
> 当异常被抛出时，具有异常安全性的函数需要满足以下两点：  
> 1、不泄漏任何资源。  
> 2、不允许数据败坏。  
> 
> 举例说明，假设有一个绘制菜单背景的类成员函数： 
> 
> {% highlight cpp %} 
> void PrettyMenu::ChangeBackground(std::istream &imgsrc)
> {
> 	lock(&mutex);				// 取得互斥器
> 	delete bgImage;				// 摆脱旧的背景图像
> 	++imageChanges;				// 修改图像变更次数
> 	bgImage = new Image(imgSrc);	// 安装新的背景图像
> 	unlock(&mutex);				// 释放互斥器
> }
> {% endhighlight %}
> 
> 该代码片段没有做到第1点，因为如果“new Image(imgSrc)”发生异常，unlock将不被执行，使得mutex被永远锁住。  
> 同样，该代码也没有实现第2点，因为如果“new Image(imgSrc)”发生异常，bgImage指向的是一个被删除的对象，imageChanges也被累加，但实际上新图像并没有安装成功。
> 
> 解决异常安全第一点“资源泄漏”问题是很容易的。  
> 我们要确保任何时候资源总能被释放，因此我们不能依靠手动保证对每一次new都调用相应的delete，即使如此，我们依然不能保证每一个new对应的delete都能得到执行。  
> 解决该问题的关键是利用C++**析构函数自动调用机制**。其主要思路是：  
> 1、获得资源后立刻放进管理对象(managing object)内。  
> 2、管理对象在离开作用域时自动调用析构函数确保资源被释放。  
> 
> 我们常用的管理对象有std::auto_ptr和boost::shared_ptr(或std::tr1::shared_ptr)。如对于上文提及例子中的lock(&mutex)，我们可以做如下修改：
> 
> {% highlight cpp %} 
> class Lock
> {
> public:
> 	explicit Lock(Mutex* pm) : mutexPtr(pm, unlock)
> 	{
> 		lock(mutexPtr.get());
> 	}
> private:
> 	std::tr1::shared_ptr<MutexmutexPtr;
> };
> {% endhighlight %}
> 
> 本小节考虑异常安全的第2点“数据败坏”问题。但是解决数据败坏问题远没有解决资源泄漏那么简单。  
> 我们可以把异常安全分为几个级别：  
> 1、None。代码不保证异常安全，代码可能有严重的资源泄漏，可能在异常发生时崩溃。  
> 2、Basic。这是我们代码应该提供的基本保证。当异常发生时，要求没有资源泄漏，所有的对象都是完整的。如类的约束条件依然获得满足，菜单的背景可以是原来的图像，也可以是缺省的图像，但客户无法预期。  
> 3、Strong。函数流程执行要么全部成功，或者发生异常，当发生异常时，所有的数据必须保持流程执行前的状态。  
> 4、Nothrow/nofail。函数流程总是会成功执行。  
> 
> {% highlight cpp %}
> void doSomething(T & t)
> {
>  t.integer += 1 ;    // 1. nothrow/nofail
>    X * x = new X() ;     // 2. basic : can throw with new and X constructor
>    t.list.push_back(x) ;  // 3. strong : can throw
>    x.doSomethingThatCanThrow() ;  // 4. basic : can throw
> }
> {% endhighlight %}
> 
> 上面的代码虽能编译执行，但不提供任何异常安全性。即处于None级别。因为如果第3步异常，x 将泄漏。  
> 
> {% highlight cpp %}
> void doSomething(T & t)
> {
>    t.integer += 1 ;                // 1.  nothrow/nofail
>    std::auto_ptr<Xx(new X()) ;  // 2.  basic : can throw with new and X constructor
>    X * px = x.get() ;              // 2'. nothrow/nofail
>    t.list.push_back(px) ;          // 3.  strong : can throw
>    x.release() ;                    // 3'. nothrow/nofail
>    x.doSomethingThatCanThrow() ;  // 4.  basic : can throw
> }
> {% endhighlight %}
> 
> 上面的代码提供basic级别的异常安全保证，无论何处发生异常，该函数均不会造成资源泄漏。  
> 
> {% highlight cpp %}
> void doSomething(T & t)
> {
>    std::auto_ptr<Xx(new X()) ;    // 1. basic : can throw with new and X constructor
>    X * px = x.get() ;               // 2. nothrow/nofail
>    x.doSomethingThatCanThrow() ;   // 3. basic : can throw
> 
>    T t2(t) ;  // 4. strong : can throw with T copy-constructor
>    t2.list.push_back(px) ;         // 5. strong : can throw
>    x.release() ;                    // 6. nothrow/nofail
>    t2.integer += 1 ;                // 7. nothrow/nofail
> 
>    t.swap(t2) ;                     // 8. nothrow/nofail
> }
> {% endhighlight %}
> 
> 面的代码提供strong级别的异常安全保证，即不会发生资源泄漏，同时函数要么成功进入新状态，要么失败保持原有状态。  
> 从代码可以看出，这里我们使用了一种“copy-and-swap”的策略。如果某处发生异常，t仍然保持原来的状态不变，因为4~7步修改的是t的副本t2，同时我们可以认为第8步swap操作总是能够成功的(nothrow/nofail 级别)。  
> 从该示例代码也可以看出，我们对t做了备份t2，最终用t2替换t，这涉及两次额外的拷贝操作，如果t数据量很大的话，这些操作带来很多额外的时间。因此，在任意情况下保持strong级别的异常安全性并不一定是实用的，要根据项目实际看是否必要。  
> NOTE：我们为什么认为第8步swap总是能成功？我认为原因是：第8步的操作都是最基本的操作，就像整数相加一样。虽然我也认为没有一行命令会永远能够成功执行(比如机器突然断电就有可能让其失败)，但是为了分析方便，我们不考虑这些不常见因素(事实上，这一点或许就是平时最困扰我们的，但我认为我们要知道：我们不可能写出任何内外部因素下都能正确执行的代码)。
> 
> 总而言之，就如希望做到bug-free一样，我们是很难写出完全exception-safe的代码，但我们总是根据项目特点综合考虑，写出符合要求的代码。  
> 可见，我想没有某个万能的工具包能够一劳永逸地解决异常安全性问题，但是本文提出的思路可以帮助我们写出更加exception-safe的代码。
