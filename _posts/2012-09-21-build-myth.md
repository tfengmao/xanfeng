---
title: 构建的迷思
layout: post
category: programming
tags: build autotools autoconf configure make pkg-config
---

*./configure && make && (sudo) make install，这一组合多么令人心旷神怡！*

autobook我很早就读过，至今仍不理解，真是一份难读的资料。[dwheeler](http://www.dwheeler.com/autotools/)认为学习autotools，不能以autobook为入口。  
好资料是良好开端，因此先列出dwheeler推荐的资料，虽不完美，但的确不错：  
1、dwheeler：[Introduction to the Autotools](http://www.dwheeler.com/autotools/)  
2、[Adventures in Autoconfiscation](http://www.jezuk.co.uk/cgi-bin/view/articles/autoconfiscation-part-one)：Arabica作者的使用心路。  
3、[Autotools Mythbuster](http://www.flameeyes.eu/autotools-mythbuster/index.html)  
4、[Building C/C++ libraries with Automake and Autoconf](http://www.openismus.com/documents/linux/building_libraries/building_libraries)  
5、[autotools tutorial](http://www.lrde.epita.fr/~adl/autotools.html)  

为了跨机器/平台，为了方便，我们亟需自动构建。自动构建非autotools一家，CMake便是时下新宠。但GNU autotools由来已久(autobook成书就是2000年)，不论众说如何纷纭，理解autotools还是必要的。

###Autofools & Autohell

网上诸多autotools的资料，但多只涉皮毛，很多看来是受了autobook的“荼毒”。国内外很多人都觉得autotools难以理解、落后，就是个灾难，称之为autofools或者autohell。据我经验，很有可能他们也是从autobook入手的。

这些文章提到的功能、性能不足，就暂且不论了。  
[Escape from GNU Autohell](http://www.shlomifish.org/open-source/anti/autohell/)  
[不再“自動變傻”](http://imtx.me/archives/1352.html)  

###autotools

所谓autotools，又称GNU build system，是指autoconf、automake、libtool三件套。

写成一个project之后，要做的事情是**编译**(compile)和**安装**(install)：  
- 项目小，依赖少，就几个源文件，径直gcc即可。  
- 项目中等，有数个源文件夹，数十个源文件，再手动编译会觉吃力烦扰，需要自动编译支持，此时手写Makefile即可。  
- 项目大，依赖多，手写Makefile也觉得烦，写出来的Makefile很大，难以维护。需要automake这样的工具来减少人力。  

更何况，不管项目大小，在机器A上能编译安装，在机器B上就不一定能编译安装，可能少了依赖库，可能依赖库的版本不合要求，可能...  
这就需要configure。configure的工作大同小异，有规可循，我们当然不想纯手工制作，autoconf就来减少这部分人力。  

autoconf提供了一族工具，没必要一一了解，autoreconf就搞定了。总之不管工具使用如何繁复，有两点始终不变：  
1、需要生成Makefile，用来编译、安装。  
2、需要指定configure哪些东西，如何编译。放到autotools里分别对应configure.ac和Makefile.am。  

这两点是根本，其余变化都围绕着两个不变点。  
同时每个工具都要遵循规则，对应的每个设定文件都需要遵循语法和规则。  

**autoconf、autoscan、pkg-config**  
configure.ac指定如何生成configure文件，指定编译前需要检查哪些东西。  
推荐写作顺序：  
{% highlight text %}
AC_INIT(package, version, bug-report-address)
information on the package
checks for programs
checks for libraries
checks for headers 
checks for types
checks for structures
checks for compiler characteristics
checks for library functions
checks for system services
AC_CONFIG_FILES([file ...])
AC_OUTPUT
{% endhighlight %}

举个栗子，gettimeofday函数一个典型的可能带来portability问题的函数(?)，就要放到configure.ac中library functions那一块。  
再举个栗子，C代码中使用了bool(不是自己定义的，而是内核或glibc某处定义的)，如果放到别的机器上编译，事先也要检查一下对方是否有bool，否则编译肯定不过。  

这些应该都是“广为人知”的portability需要考虑的点，你仔细看代码+回忆，看自己到底用了哪些，然后一一写出。  
...总觉不爽利，人肉检查慢，而且容易遗漏，有没有工具帮忙？  

对于这么一个有规律性的任务，当然有工具：autoscan。autoscan的原理也就好理解了：  
其中最基础的一步，是根据预定义的一个“checklist”，使用autoscan扫描project文件夹，生成GNU M4格式的configure.scan。这个checklist是/usr/share/autoconf/autoscan/autoscan.list，其内容大致为:  
{% highlight text %}
function: getspnam AC_CHECK_FUNCS
function: gettimeofday AC_CHECK_FUNCS
function: getusershell AC_CHECK_FUNCS
function: getwd warn: getwd is deprecated, use getcwd instead
...
header: OS.h            AC_CHECK_HEADERS
header: X11/Xlib.h              AC_PATH_X
header: alloca.h                AC_FUNC_ALLOCA
header: argz.h          AC_CHECK_HEADERS
header: arpa/inet.h             AC_CHECK_HEADERS
header: fcntl.h         AC_CHECK_HEADERS
...
identifier: bool                AC_CHECK_HEADER_STDBOOL
identifier: false               AC_CHECK_HEADER_STDBOOL
identifier: gid_t               AC_TYPE_UID_T
identifier: inline              AC_C_INLINE
...
makevar: AWK            AC_PROG_AWK
makevar: BISON          AC_PROG_YACC
makevar: CC             AC_PROG_CC
...
program: CC             AC_PROG_CXX
program: awk            AC_PROG_AWK
program: bison          AC_PROG_YACC
program: byacc          AC_PROG_YACC
...
{% endhighlight %}

configure.ac中更重要的一块是，检查项目依赖的库是否存在。这是通过[AC_CHECK_LIB、AC_SEARCH_LIBS](http://www.gnu.org/software/hello/manual/autoconf/Libraries.html)实现的，不过更好的做法是使用pkg-config.

pkg-config用来找库，系统每安装一个库，如果遵循规则，就会往某处写一个.pc文件，里面包含库信息、库的依赖和库使用指南。pkg-config就是根据这些.pc文件来管理库的，所以这些
http://blog.csdn.net/absurd/article/details/599813  
http://www.chenjunlu.com/2011/03/understanding-pkg-config-tool/  
http://people.freedesktop.org/~dbn/pkg-config-guide.html  

**automake**  
检查之后，就是编译。需要提供Makefile.am，automake据此生成Makefile。  

整体流程是这样的：  
![](/images/autotools_flow.PNG)  

**libtool**  

其他暂时没什么好说的，看个例子吧：https://github.com/jezhiggins/arabica