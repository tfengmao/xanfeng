---
title: 动态连接库和符号(symbol)
layout: post
tags: so library shared symbols elf ldconfig
category: linux
---

###shared library (.so)

"[Program Library Howto-Shared Libraries](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)"是很好的材料, 下面的内容多是据此整理的.

**定义**:  
Shared libraries are libraries that are loaded by programs when they start.  
使用shared library(共享库)会有很多好处, 比如软件升级, 不难想象.

**命名约定**:  
1. soname: 每个共享库都有一个soname, 形式为"libNAME.so.x", 其中x是版本号. 如"libc.so.6".  
2. real name: 真正的库文件, 一般形式为"soname.y[.z]", 即"libName.so.x.y[.z]", 其中y是minor number, z是release number, 是可选的. 如"libattr.so.1.1.0".  
3. linker name: compiler用来请求库时使用的名字, 一般是没有版本号的soname.    

**放置位置 & load/preload**:  
共享库一般放在一些约定的目录下, 如/usr/lib/, /usr/local/lib, /lib/等. 这其实是遵循FHS的, 比如/usr/local/lib下放置的一般是用户开发的库.  
在启动程序时, program loader(ld-linux.so.x)会找到并加载程序需要的共享库, loader查找的路径一般就是上述的几个目录, 这些目录在**/etc/ld.so.conf**文件中配置.  
如果只想覆盖共享库的某几个函数, 保持其余函数不变, 则可以将共享库名字和函数名字输入到/etc/ld.so.preload中, 这里面定义的规则会覆盖标准规则.  

**cache arrangement & ldconfig**  
实际上, 在启动程序时再去搜寻所需的共享库不是高效做法, 所以loader使用了cache. ldconfig的作用就是读取文件/etc/ld.so.conf, 在各个库目录中, 对共享库设置合适的symbolic link(使得遵守命名约定), 然后写入某种数据到/etc/ld.so.cache, 这个文件再今后就被其他程序使用, 从而大幅提升了共享库的查找速度.  
所以在每加入/移除一个共享库, 或者修改了/etc/ld.so.conf(即修改库目录)的时候, 最要运行ldconfig.

**创建共享库**  
step1. 编译出object files, 需要使用***-fPIC***或***-fpic*** flag. fPIC和fpic的区别是, 前者生成的文件更大, 不过具有更好的平台无关性, 后者恰好相反. 这说明前者为了platform-independence做了更多工作.  
step2. 用-Wl向linker传递参数. 如: "gcc -shared -Wl,-soname,libmystuff.so.1 -o libmystuff.so.1.0.1 a.o b.o -lc".  
step3. 把共享库拷贝到约定的某个目录下即可, 如/usr/local/lib.  
step4. ldconfig -n /path/to/lib.  

###elf

elf的内容参考"[elf & libelf, elftoolchain](http://xanpeng.github.com/2012/05/17/elf-libelf/)", 它是一种格式，也是一种规范, 可以用libelf写程序去操作它, 可以用objdump、nm和readelf去读取elf文件的内容.  

###symbols

我也已经熟悉共享库了, 我知道ldconfig的作用, 我知道常用的库放置目录, 我知道ltrace, ldd可以用来帮助确认某程序和某些共享库的关联关系是否正确.  
所以, 如果没有**symbols**这一节, 本篇文章存在的意义不大.

"[Inside ELF Symbol Tables](https://blogs.oracle.com/ali/entry/inside_elf_symbol_tables)"是绝佳的资料, 当然正如很多网文一样, 它仅是帮助理解, 而不涉及很深的细节. 细节标准什么的还是要看书和文档了, 这方面很不错的书籍就是校友的<程序员的自我修养>了.

查看elf规范, 你必然可以看到symtab和dynsym, 如"[ELF-64 Object File Format](http://downloads.openwatcom.org/ftp/devel/docs/elf-64-gen.pdf)"中"4.Sections"就列出了标准的sections, .symtab和.dynsym就是其中之二.  
实际上, 我们知道机器可执行的是machine code, 而我们使用的高级语言编程, 并不是利用晦涩的机器码, 而是用human-readable的变量名, 函数名等, 这些名字就是**symbolic name**. 编译器在编译时收集symbol信息, 并储存在object file的.symtab和.dynsym中. symbols是**linker**和**debugger**所必需的信息, 如果没有symbols, 试想debugger如何能展示给用户调试信息了? 如果没有symbol, 而只有地址(相对于object file的offset), linker如何能够链接多个object file了?  
对于linker和symbol, 我们可以做个小实验:

{% highlight c %}
// 编写一个简单的 a.c
$ cat a.c
void func(void)
{
		printf("call func()\n");
}

$ nm a.o
00000000 T func
		 U puts

// 编写一个简单的 main.c
$ cat main.c
#include <stdio.h>
extern void func(void);
int main(
{
		func();
		return 0;
}

$ nm main.o
		 U func
		 00000000 T main

// 正常情况下
$ gcc main.o a.o -o main
$ ./main
call func()

// 为了验证symbol对于linker来说是必需品, 我做如下操作
$ file a.o
a.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
$ strip a.o
$ file a.o
a.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), stripped
$ gcc main.o a.o -o main
main.o: In function `main':
/home/xan/lab/main.c:7: undefined reference to `func'
collect2: ld returned 1 exit status
{% endhighlight %}

这个小实验证实了symbols对于linker的重要性, 同时使用file看出"not stripped"->"stripped"的变化, 说的就是去除了symbols信息.

现在假使我们生成了最后的可执行文件(当然是elf格式了), 那么这个elf中是否包含symbols呢? 其中又是否需要symbols呢?  
不妨先下结论: 一般地, 生成的可执行文件都是包含symbols, 不过这些信息不是程序执行所必需的, 可以通过`strip`(Discard symbols from object files)去除.  
同样可以做个小实验:

{% highlight text %}
// 仍用上面实验的代码
$ file main
main: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.18, not stripped
$ ./main
call func()
$ strip main
$ file main
main: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.18, stripped
$ ./main
call func()
$ 
{% endhighlight %}

这个小实验证实了symbols对于可执行文件来说不是必需的, 这是因为可执行的代码都是machine code, 只需要address信息, 无需symbol信息.

对于elf和symbols, 还是好理解的啦. 就是我elf文件中留了一席之地给你放symbols, compiler在生成elf时会往其中填充. debugger/nm/readelf等可以来读取. 不过这些symbols不是程序执行必需的, 所以完全可以去除, 只不过去除之后, debugger就读不到信息了.  

而对于共享库来说, 情况略复杂些了. 我们来特别说明.  

**共享库和symbols**  

在继续下去之前, 先来看两个事实.

{% highlight text %}
$ ldd /bin/ls
		linux-gate.so.1 =>  (0xb7711000)
		libselinux.so.1 => /lib/libselinux.so.1 (0xb76e5000)
		librt.so.1 => /lib/i686/cmov/librt.so.1 (0xb76dc000)
		libacl.so.1 => /lib/libacl.so.1 (0xb76d4000)
		libc.so.6 => /lib/i686/cmov/libc.so.6 (0xb758d000)
		libdl.so.2 => /lib/i686/cmov/libdl.so.2 (0xb7589000)
		/lib/ld-linux.so.2 (0xb7712000)
		libpthread.so.0 => /lib/i686/cmov/libpthread.so.0 (0xb7570000)
		libattr.so.1 => /lib/libattr.so.1 (0xb756b000)
$ nm /lib/i686/cmov/libc.so.6 
nm: /lib/i686/cmov/libc.so.6: no symbols    --> libattr, libacl也一样, 都显示"no symbols"

// 而libpthread有一大串
$ nm /lib/i686/cmov/libpthread.so.0 | tail
		U twalk@@GLIBC_2.0
		U uname@@GLIBC_2.0
		U unlink@@GLIBC_2.0
0000c930 t unwind_cleanup
0000c970 t unwind_stop
0000cfd0 W vfork
0000de90 W wait
0000df50 W waitpid
0000c140 t walker
0000d020 W write
{% endhighlight %}

这仅仅是因为有些库做了strip, 而其他库没做strip而已? 还是说对于某些共享库来说, symbols也是必需的?  
目前不知答案, 分析下去.

前面提到.symtab和.dynsym两个不同的symbol table, 它们有什么区别?  
.dynsym是.symtab的一个子集, 大家都有疑问, 为什么要两个信息重合的结构?  
需要先了解**allocable/non-allocable ELF section**, ELF文件包含一些sections(如code和data)是在运行时需要的, 这些sections被称为**allocable**; 而其他一些sections仅仅是linker,debugger等工具需要, 在运行时并不需要, 这些sections被称为**non-allocable**的. 当linker构建ELF文件时, 它把allocable的数据放到一个地方, 将non-allocable的数据放到其他地方. 当OS加载ELF文件时, 仅仅allocable的数据被映射到内存, non-allocable的数据仍静静地呆在文件里不被处理. strip就是用来移除某些non-allocable sections的.  
.symtab包含大量linker,debugger需要的数据, 但并不为runtime必需, 它是non-allocable的; .dynsym包含.symtab的一个子集, 比如共享库所需要在runtime加载的函数对应的symbols, 它世allocable的.  

因此, 得到答案:  
1. strip移除的应是.symtab.  
2. nm读取的应是.symtab: 上面发现的libattr等nm结果为空, libpthread nm结果非空应是正常的.
3. 共享库包含的.dynsym是runtime必需的, 是allocable的.  

可做验证, 期望的结果为:  
1. strip libpthread, ls依然能够工作.  
2. strip libpthread, nm libpthread得到结果为空.  
3. 可以通过设置nm options, 或使用readelf读出.dynsym的内容.

{% highlight text %}
$ sudo strip /lib/i686/cmov/libpthread-2.11.3.so
$ file /lib/i686/cmov/libpthread-2.11.3.so
/lib/i686/cmov/libpthread-2.11.3.so: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.18, stripped
$ ls
...(输出正确结果)

$ nm /lib/i686/cmov/libpthread-2.11.3.so
nm: /lib/i686/cmov/libpthread-2.11.3.so: no symbols

$ readelf -s /lib/i686/cmov/libpthread-2.11.3.so | tail
322: 0000b6a0   292 FUNC    GLOBAL DEFAULT   13 __pthread_clock_gettime@@GLIBC_PRIVATE
323: 0000ec30    46 FUNC    GLOBAL DEFAULT   13 pthread_mutex_consistent_@@GLIBC_2.4
324: 0000b3a0    50 FUNC    GLOBAL DEFAULT   13 pthread_testcancel@@GLIBC_2.0
325: 0000d6b0   111 FUNC    WEAK   DEFAULT   13 fsync@@GLIBC_2.0
326: 0000d1f0   180 FUNC    WEAK   DEFAULT   13 fcntl@@GLIBC_2.0
327: 0000dde0   176 FUNC    WEAK   DEFAULT   13 tcdrain@@GLIBC_2.0
328: 00009390     7 FUNC    GLOBAL DEFAULT   13 pthread_mutexattr_destroy@@GLIBC_2.0
329: 00006de0    23 FUNC    GLOBAL DEFAULT   13 pthread_yield@@GLIBC_2.2
330: 000077c0   259 FUNC    GLOBAL DEFAULT   13 pthread_mutex_init@@GLIBC_2.0
331: 000093c0    49 FUNC    GLOBAL DEFAULT   13 pthread_mutexattr_setpsha@@GLIBC_2.2

$ readelf -s /lib/libattr.so.1.1.0 | tail
48: 00002f50    50 FUNC    GLOBAL DEFAULT   13 lremovexattr@@ATTR_1.0
49: 00003010    57 FUNC    GLOBAL DEFAULT   13 llistxattr@@ATTR_1.0
50: 00002ae0    50 FUNC    GLOBAL DEFAULT   13 attr_copy_check_permissio@@ATTR_1.1
51: 00001b50   259 FUNC    GLOBAL DEFAULT   13 attr_set@@ATTR_1.0
52: 00002b20  1002 FUNC    GLOBAL DEFAULT   13 attr_copy_action
53: 000031f0    71 FUNC    GLOBAL DEFAULT   13 setxattr@@ATTR_1.0
54: 00001380   543 FUNC    GLOBAL DEFAULT   13 attr_list@@ATTR_1.2
55: 000030d0    64 FUNC    GLOBAL DEFAULT   13 lgetxattr@@ATTR_1.0
56: 00002fd0    57 FUNC    GLOBAL DEFAULT   13 flistxattr@@ATTR_1.0
57: 00002f10    50 FUNC    GLOBAL DEFAULT   13 fremovexattr@@ATTR_1.0
{% endhighlight %}

至此, 对symbols和共享库,ELF的关系的了解告一段落.

###more

既然已经说到共享库(shared library), 不妨稍微提一下动态装载库([Dynamically Loaded Libraries](http://tldp.org/HOWTO/Program-Library-HOWTO/dl-libraries.html)), 共享库是在程序startup时被加载, 而DLL(注意区别于windows下的概念)则是在程序运行过程中显式被加载, 实际上就是调用dlopen,dlsym等接口显式地打开共享库, 显示地查找库中的symbol, 然后找到对应的代码去执行.

[How To Write Shared Libraries](http://www.akkadia.org/drepper/dsohowto.pdf) by Ulrich Drepper.
