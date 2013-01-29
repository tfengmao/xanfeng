---
title: ONE Sunstone-Client浅析
layout: post
category: cloud
tags: sunstone ruby javascript jquery
---

怒了，由于别人非常不给力＋Copy＆Paste的恶劣代码风格，我忍不下去了，我决定亲自动手！  
虽然这肯定会浪费我宝贵的时间，但没办法，在现状下要完成任务，这部分时间是必然要被浪费的。  

除了六七年前本科的“Hello World”级别的web项目，我几乎从来没有做过前台，因此我势必不能把控全局，但所幸ONE的页面不是很复杂的样子，只要知晓了其流程和思路，再利用语言无关的程序化思维和已有参照，一定能搞定新页面功能的开发。  

先来说ONE页面的特点。  
1、ONE的页面前端、服务端功能都在Sunstone项目中实现，应该分别称之为Sunstone-Client和Sunstone-Server。当然本文只关注Sunstone-Client，和Server端关系不大。  
2、ONE sunstone-client使用了大量的JavaScript代码，并且使用了JQuery。  
3、HTML代码都是写在JavaScript里面的，由JS控制什么时候显示那一部分。  
4、页面加载时，会一次发出多个HTTP请求，并从Server获得数据，点击tab时，触发JS代码，显示已经装载好数据的对应页面。  

再来补足一些基础的JS知识和JQuery知识（个人理解，可能有误）。  
1、用了很多的`$(document).ready()`，这是JQuery提供的功能，用于在页面DOM装载完成后，执行其后定义的function。参考“[Tutorials:Introducing $(document).ready()](http://docs.jquery.com/Tutorials:Introducing_$(document).ready())”。  
2、JS代码就像Python等脚本语言的代码一样，是没有main函数的，源文件里的函数调用和变量定义都被按序执行。参考“[Why doesnot JavaScript need a main( function?)](http://stackoverflow.com/questions/9015836/why-doesnt-javascript-need-a-main-function)”、“[How do browsers execute javascript](http://stackoverflow.com/questions/5317190/how-do-browsers-execute-javascript)”。  
3、使用了很多匿名函数，参考“[Javascript的匿名函数](http://dancewithnet.com/2008/05/07/javascript-anonymous-function/)”。  

再来看Sunstone-Client页面实现的逻辑，假设要新增一个hello页面：  
1、在/public/js/plugins/下面增加hello-tab.js。  
2、在hello-tab.js里面编写`$(document).ready(function({...})`，比如我想加载页面时显示一个列表，那么这里面要调用Sunstone.runAction("hello.list")。  
3、action是通过Sunstone.addActions(helloActions)添加的，addActions是Sunstone-Client提供的util函数，helloActions是我们自己定义的变量，其中包含了action的实体，如下例所示：  

{% highlight javascript %}
"hello.list" : {
	type: "list",
	call: function() {
		$.ajax( {
			async: false,
			type: "GET",
			timeout: 45000, 
			dataType: 'jsonp',
			jsonp: 'jsonpCallback',
			url: constant.getHelloListUrl,
			success: function(json) {
				if(json[0].success) {
					updateHelloView(json[0].result);
				}
			},
			error: function(response) {
				alert(tr('Bind VM with EIP failed!'));
			}
		});
	}
},
{% endhighlight %}

4、上面的action是页面想服务端发送请求的地方。要显示页面，需要调用Sunstone.addMainTab(helloTab)完成的，helloTab是一个变量，它定义了hello页面的模块，如：  

{% highlight javascript %}
var helloTab = {
	title: tr("Hello Tab"),
	content: helloTabContent,
	buttons: helloButtons,
	tabClass: 'subTab subsunTab',
	parentTab: 'someParentTabVar'
};
{% endhighlight %}

5、content是hello tab页面的主体内容，buttons是页面上的按钮集合，ONE的按钮都默认在右上角。这两部分内容分别用对应的变量指定，变量中都会嵌入HTML页面元素。  

至此，ONE Sunstone-Client页面的逻辑就清楚了。初始页面是这么加载的，请求是这么发送的，请求对应的页面是这么调出来的。  
另外，还会有其他一些细节，大多是遵循Sunstone-Client自定义规约的一些细节，其用途在Sunstone对应的JS源码里面都有注释说明。
