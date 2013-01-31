---
title: JavaScript进阶学习
layout: post
category: coding
tags: javascript function clousure
---

我已经迫不及待地要写下这篇博文了。  
正如上篇博文说到的，我不堪忍受现状，决定入主页面开发。发现开发中面对大量的JavaScript代码，我是参考着现有的页面做新的开发，结果看到大量重复的JS代码，大量类似的功能没有被抽象成公共的函数，大量的全局变量使得命名成了一大问题。  
我想，肯定是可以写得更好的！于是我开始阅读那本著名的图书“[JavaScript: The good parts](http://book.douban.com/subject/2994925/)”(值得多次阅读的书！)。结果是书中的内容让我激动，它展示了JavaScript是一个功能强大的、有大量优点和一些设计缺陷的、面向对象的函数式编程语言。  
本文记录一些重点，同时我创建了[一个repo](https://github.com/xanpeng/javascript-recipes)，会在上面放置一些示例代码和留待后用的代码。  

在开始之前要特别声明：我的核心工作将仍然是分布式后台，此次JavaScript学习就当是一项兴趣之旅，我决不会+决不能借此转入前端！

###语法

JavaScript的语法十分接近C的语法，不同之处是它是弱类型的、是函数式的、是面向对象的。  

下面列出一些小细节：  
1 编译单元：一个JS源文件。 
2 对象不是基于Java之类的类型体系，而是基于原型。原型连接只有在检索时才被用到。  
3 对象通过引用传递。  
4 函数也是对象，也是变量。  
5 当一个函数被保存为对象的一个属性时，称之为方法。  
6 **闭包(closure)**：函数可以访问被创建时的上下文环境。  
7 apply可以用来“盗用”其他函数或方法。  
8 函数总是返回一个值：undefined或其它指定值。  
9 有函数作用域，但没有块级作用域。  
11 "||"可用以填充默认值：var status = flight.status || "unknown"。  

###代码风格+最佳实践

1 使用K&R风格。  
2 "{"放在一行的结尾，可避免return语句中一个可怕的设计错误。  
3 用"//"表示注释。 
4 用一个单独的全局变量包含一个脚本应用或工具库，用以模拟命名空间。使用闭包更好地模块化。  
5 在每个函数的头部声明所有变量。  
6 使用函数表达式，而不是function语句。  

{% highlight javascript %}
// 函数表达式
var foo = function foo() {...};
// function语句，在解析时会发生被提升的情况：不管放置在何处，都会被移动到被定义时所在作用域的顶层。
// 这放宽了函数必须先声明后使用的要求，而"我"认为这会导致混乱。
function foo() {...}
{% endhighlight %}

7 尽量不去使用new。  
8 优先使用"."检索对象。  


###糟粕和鸡肋

1 全局变量，隐式全局变量(未经声明的)。  
2 有类似于C语言的代码块(一对花括号包含)，但却没有对应的块级作用域，因此经常让人吃惊。  
3 自动插入分号，看示例：  

{% highlight javascript %}
// 下面的return语句中，return后面会被自动插入分号，
// 从而使得：意图是返回一个对象字面量，结果是"return;"返回undefined。
return
{
	status : true
};

// 因此要写成：
return {
	status : true
};
{% endhighlight %}

4 过多没有用到的保留字。  
5 Unicode范围增长：65536~百万级别。JavaScript却是在65536的时候设计的。  
6 假值太多。  
7 hasOwnProperty是一个方法，而不是一个运算符(函数)，因此在任何对象中，它可以被赋予一个错误/不合适的值。  
8 JavaScript中永远不会有真的空对象，因为它们可以从原型链中取得成员元素。  
9 "=="和"!="是邪恶的，只有"==="和"!=="才符合你的预期：比较类型和值。  
10 结果不可预料的with语句。  
11 eval，用以传递一个字符串给JavaScript编译器执行。这是一个被滥用的特性，“尤其是那些对JavaScript语言一知半解的人们经常用到它”。它不便阅读，因为要运行编译器而使性能显著降低，而且安全性差。Function构造器是eval的另一种形式，所以它同样要被避免使用。  
12 function语句，对比函数表达式。  
13 new运算符可能引起的误用。  

###JSON

JSON有6种类型的值：对象、数组、字符串、数字、布尔值(true, false)和特殊值null。  
JSON对象是一个key-value对的无序集合，key可以是任何字符串，value可以是上述6中类型的任意一种。  

JSON使用：不用eval，也不用XMLHttpRequest(只能从生成HTML的同一服务器获取数据)，而是用[JSON.parse](https://github.com/douglascrockford/JSON-js)。

###[JQuery](http://www.w3schools.com/jquery/default.asp)

features:  
1 HTML/DOM操作。  
2 CSS操作。 
3 HTML event methods.  
4 effects & animations.  
5 AJAX.  
6 Utilities.  

语法：$(selector).action()  
* A $ sign to define/access jQuery.  
* A (selector) to "query (or find)" HTML elements.  
* A jQuery action( to be performed on the element(s))  

语法示例:  
`$(this).hide()` - hides the current element.  
element selector: `$("p").hide()` - hides all <p> elements.  
.class selector: `$(".test").hide()` - hides all elements with class="test".  
\#id selector: `$("#test").hide()` - hides the element with id="test".  
组合选择：  
![](/images/jquery_selector.png)  

DOM操作：  
1 get/set content - text(), html(), val().  
* text() - Sets or returns the text content of selected elements  
* html() - Sets or returns the content of selected elements (including HTML markup)  
* val() - Sets or returns the value of form fields  
2 get/set attributes - attr().  
3 add elements - append(), prepend(), after(), before().  
4 remove elements - remove(), empty().  
5 dimension methods(*图片引自w3schools*):  
![](http://www.w3schools.com/jquery/img_jquerydim.gif)  

[jQuery AJAX](http://www.w3schools.com/jquery/jquery_ref_ajax.asp)([Asynchronous JavaScript and XML](http://www.w3schools.com/jquery/jquery_ref_ajax.asp)): exchanging data with server, and updating parts of a web page.  
1 load() - `$(selector).load(URL, [data], [callback])`，从server获取数据，加载到选定的页面元素中。  
2 [get() & post()](http://www.w3schools.com/jquery/jquery_ajax_get_post.asp)  

