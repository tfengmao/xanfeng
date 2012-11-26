---
title: 内核链表(linux/list.h)
layout: post
category: coding
tags: linux macro offset typeof
---

###container_of

内核链表（include/linux/list.h）是可以用在用户态的，因为它全是宏定义和inline函数。它独立封装了通用的链表操作方法，然后可以结合不同的链表数据使用。  
实现这个抽象的核心是container_of宏，  
{% highlight c %}
#define container_of(ptr, type, member) ({					\
    const typeof( ((type *)0)->member ) *__mptr = (ptr);		\
    (type *)( (char *)__mptr - offsetof(type, member) );})
};

// 示例结构
struct example_struct {
	int vala;
	struct example_member member_t;
	int valb;
};
// 示例函数
struct example_struct *some_func(struct example_member *target_ptr)
{
	return container_of(target_ptr, struct example_struct, member_t);
}
{% endhighlight %}

很多网文分析了这个宏，比如[这个](http://senghoo.com/229.html)就很不错。  
其原理很简单：  
1、第一行的typeof操作符是GCC编译器指定的，获取操作数的类型。  
2、第一行的((type *)0)->member是一个小技巧，搭配typeof可以定位到结构成员member的类型。  
3、第一行定义__mptr指针的意图：有人猜测是保存ptr，以防止外部被更改，我觉得另一个原因是利用**类型检查**。  
4、第二行表示：将ptr减去member在结构体中的偏移，就得到了结构体的指针了。  
5、第二行的offsetof是另一个宏，定义位于linux/stddef.h，内容：  
{% highlight c %}
#ifdef __compiler_offsetof
#define offsetof(TYPE, MEMBER) __compiler_offsetof(TYPE,MEMBER)
#else
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#endif
{% endhighlight %}

###list_head

内核链表是双链表，list_head的定义和初始化：  
{% highlight c %}
struct list_head {
	struct list_head *next, *prev;
};

#define LIST_HEAD_INIT(name) { &(name), &(name) }
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
{% endhighlight %}

如其他链表一样，支持插入、删除、合并、遍历等操作。看一个简单示例便清楚。  
{% highlight c %}
#include "list.h"	// https://github.com/xanpeng/libraries-misc
#include <stdio.h>
#include <stdlib.h>

typedef struct data {
    int elem;
    struct list_head list;
} data_t;


LIST_HEAD(head_1);
LIST_HEAD(head_2);

void create_list(struct list_head *head, int start, int finish)
{
    int i;
    for (i = start; i <= finish; ++i) {
        data_t *d = malloc(sizeof(data_t));
        d->elem = i,
        list_add_tail(&d->list, head);  // queue
        // list_add(&d->list, head);    // stack
    }
}

void print_list(struct list_head *head)
{
    struct list_head *iter;
    data_t *dptr;

    list_for_each(iter, head) {
        dptr = list_entry(iter, data_t, list);
        printf("%d->", dptr->elem);
    }
    /*
    list_for_each_entry(dptr, head, list)
        printf("%d->", dptr->elem);
    */
    printf("\n");
}

int main()
{
    create_list(&head_1, 1, 5);
    print_list(&head_1);

    create_list(&head_2, 6, 10);
    print_list(&head_2);

    // list_move_tail(&head_2, &head_1);
    list_splice(&head_1, &head_2);
    print_list(&head_1);

    return 0;
}
{% endhighlight %}

###list_for_each_safe

多了一个临时变量，用来存储当前节点。

{% highlight c %}
/**
 * list_for_each_safe - iterate over a list safe against removal of list entry
 * @pos:	the &struct list_head to use as a loop cursor.
 * @n:		another &struct list_head to use as temporary storage
 * @head:	the head for your list.
 */
#define list_for_each_safe(pos, n, head) \
	for (pos = (head)->next, n = pos->next; pos != (head); \
		pos = n, n = pos->next)

#define list_for_each(pos, head) \
	for (pos = (head)->next; prefetch(pos->next), pos != (head); \
		pos = pos->next)
{% endhighlight %}

###[hlist](http://www.ibm.com/developerworks/cn/linux/kernel/l-chain/#N101F3)

> 精益求精的设计者（因为list.h没有署名，所以很可能就是Linus Torvalds）认为双头（next、prev）的双链表对于HASH表来说"过于浪费"，因而另行设计了一套用于HASH表应用的hlist数据结构——单指针表头双循环链表：  
![](/images/list-hlist.gif)

###RCU

在Linux链表功能接口中还有一系列以"_rcu"结尾的宏，与其他接口类似。