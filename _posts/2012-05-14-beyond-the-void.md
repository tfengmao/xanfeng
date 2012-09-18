---
title: 牛人推介-Beyond The Void 和感悟
layout: post
tags: byv kernel development intelligence english quiet learn skill interests
category: career
---

每日上午例行阅读 google reader, 了解其他人(多是coder)正在做什么, 今日通过一篇博文了解到一个牛人.

博文"[APIO讲稿——函数式编程](http://www.byvoid.com/blog/)"讲了我"神往"已久的函数式编程, 所以没有像阅读IT 明星老赵的"[编写一个“绑定友好”的WPF控件](http://blog.zhaojie.me/2012/05/wpf-binding-friendly-user-control.html)"那样直接略过了. 认真阅读了之后, 发现作者的思路清晰, 讲解简单深刻, 还结合了理论分析(他对停机问题的解释让我这个计算理论菜鸟也能很快理解), 瞬间认为这是个牛人! 根据其博客资料仰慕到更多:  
1. 郭家宝, 清华计算机大二 -- [About](http://resume.byvoid.com/)  
2. 高中开始搞 ACM 竞赛 -- 好像搞这个的多是大牛.  
3. 现在 M$ Research Asia 实习, 做分布式系统和数据库 -- [resume](http://resume.byvoid.com/home/resume)  
4. 参与多个项目, 其中有名的是 ibus-pingyin, 这个我也在用(发现不是很好用, 于是切到 [ibus-googlepinyin](http://linuxtoy.org/archives/ibus-googlepinyin.html) 了.)  
5. 写书 "Node.js: Getting Start and Practice", 即将出版.  
6. 繁体字爱好者, [OpenCC](http://code.google.com/p/opencc/) 的发起者和 Contributor.  

仰慕牛人! 在加上近期亲身接触的清华人来说, 他们的特点是:  
1. 聪明. -- 我经常在做损伤脑力的事情: 高强度体育活动, 不健康的生活习惯(如晚睡).  
2. 精力旺盛. -- 我近期身体状态绝佳, 昨天打球扭脚, 十几分钟后恢复, 然后继续打. 近期因为不健康的生活习惯, 精力状态不佳.  

反省自己, 决定调整自己的状态, 细节如下:  
1. 早睡早起, 1:00必须上床.  
2. 健康饮食.  
3. 阅读书籍 > 观看视频.  
4. 不做无聊的事.  
5. 和朋友去骑行.  

---

byvoid 的"[有幸加入ibus-pinyin的开发](http://www.byvoid.com/blog/join-develop-ibus-pinyin/)" 提到 [ibus](http://code.google.com/p/ibus/) 的作者 Peng Huang, 放狗没有找到 Peng Huang 的博客, 仅发现这篇介绍 iBus 的 [slide](http://2008.gnome.asia/static/upload/event_file/ibus-GNOME.pdf). 原来 Peng Huang 也**曾**在 redhat(I18N team). 他的资料:  
1. 曾经在 RedHat I18N.  
2. 现在 Google.   
3. [linkedin](http://ca.linkedin.com/pub/peng-huang/10/752/270)  
4. [github](https://github.com/phuang), 几乎全是 ibus 相关的代码.  

由 Peng Huang 又链接到 [Yong Sun](http://yongsun.me/about/), ibus-pinyin contributor, [sunpinyin](http://code.google.com/p/sunpinyin/) lead contributor. 好像 ibus-sunpinyin 比 ibus-pinyin 更好用, 但我使用的时间都不长, 因为对 googlepinyin 的喜爱而快速切到 ibus-googlepinyin 了. *多扯远一步: [ibus-googlepinyin](http://linuxtoy.org/archives/ibus-googlepinyin.html) 远远不如 win 下的 [Google 拼音输入法](http://www.google.com/intl/zh-CN/ime/pinyin/), 要么就是 Android 下的 Google Pinyin 很弱, 要么就是 [shellexy](twitter.com/shellexy) port 得不好, 应该是前者吧.*

---

不论 byvoid, Peng Huang 还是 Yong Sun, 他们都不是做 kernel 的. 我曾经与 @鸦片鱼 说过我对 kernel 开发的畏惧, 今天我又产生了这种畏惧. 

畏惧是好事, 有畏惧才能去想解法. 我的想法:  
1. 我对 kernel 的诸多知识很有兴趣, 我会去继续了解, 在某些方面(如 pthreads, 多线程, parallel, physical/virtual memory management)我会了解得深一些, 在其他方面我会了解得浅一些.   
2. 找机会跳出来, 或者说跳进去(进到一个可以安心做很长时间 kernel 开发的地方).  

警戒自己: **我才开始如 kernel 开发之门, 很多需要学习!**

今天让我生这种畏惧的更多是因为这篇文章: "[半个系统软件工程师的困惑](http://www.tektalk.org/2012/01/20/%E5%8D%8A%E4%B8%AA%E7%B3%BB%E7%BB%9F%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B%E5%B8%88%E7%9A%84%E5%9B%B0%E6%83%91/)", 摘引总结部分:

> 抱怨了这么多，感觉走系统软件工程师这条路，实在太辛苦了，学的东西多，发展又慢，还需要给别人擦屁股，现在不知道何去何从。看着做业务的，个个上升的很快，实在是羡慕嫉妒恨啊。之前分析问题时，也浏览过部分业务软件，就是个大case啊，解析消息，回应答，心有不甘啊。  
> 1，系统软件（偏向于OS方向）有多大的发展前途，感觉就只能在可靠性，可维护性，适配上下功夫。  
> 2，去互联网公司（如百度，QQ，假设能去的话）整系统软件会比在H公司更有前途吗？  
> 3，转行做APP呢？  
 
下面人们各种回复, 主要是以下方面的内容:  
1. 赞叹楼主学的很多, 优化过 TCP/IP 协议栈(*这里我很有疑惑, 有多少人能去合理优化 TCP/IP 协议栈, 什么场景下才不得不去优化协议栈?*)  
2. 技术!=事业. 做技术不能钻牛角尖, 要找到自己的位置.  
3. 去一个好的合适的公司.  

强调 [beans](http://www.beanos.org/) 的回复:

> 兄弟，我做系统也有8年多了，我也曾经有过你这样的困惑。为什么我拼死拼活的解决系统疑难问题，提高系统可靠性，最后却被忽视掉~。  
> 但是后来我发现，这也许不是自己的问题，而是没有找到合适的位置。  
> 并不是哪个公司都重视有技术天赋的员工，并给你机会的，毕竟解决系统问题不能给公司直接带来人民币。  
> 你们H的情况就更复杂了，我有不少认识的同学同事，技术都很不错，可是死活在H就是没得混，但是在别的公司，却是都干得相当好。你实在不行就跑路吧，需要你这样的人才的地方多去了，没必要委屈自己。  

> 首席这么夸我，我都不好意思了。系统这方面，在这里，我也只能算是个票友。  
> 我有个很好的朋友，他在共享软件联盟里混了很久，就是一帮写共享软件卖钱的人，他加入了一个小公司，他的老板是跟我一同毕业的，人家写了个杀毒的软件，几个人的小公司，每年有几千万的入账，好几年前，就已经把公司总部搬到美国去了。要说技术什么的，其实大家都差不多，只不过是选择的路不一样，最后到达的地方就会不同。  
> 我看了首席的发言，感觉首席心里也很挣扎，但是我非常理解，或者每个做这个活的程序员，都会有这样的挣扎，其实我也挣扎过，我做过几个半成品软件(比如差不多能用的手写识别，就比汉王那个稍微差一点，嘿嘿)，但终究没有勇气放弃这一行，毕竟这方面的工作带给我稳定的收入，并且也担负着一定的责任。

强调 westermann 的回复:

> 要贡献！贡献！总是说花了n年研究linux内核神马的, 没在社区留名就啥也不是! 看看人家robert love 80年的小伙儿曾几何时就在社区留下了O(1)调度器外加一本kernel development的书给自己挣足了名气. 不然做再多也是无用功.   
> 别跟我谈不在乎名利, 不然也不会发出这种英雄无用武之地的感慨.  
> 不过话说回来, linux kernel这个名利场早就不剩下什么汤了，该被占的坑已经占得差不多了, 选错了战场怎么打都是输.

总结:  
1. 每个人都有不同的评价, 或从待遇方面(实), 或说要是怎么...如果怎样...那就怎样...(虚).  
2. 实际上很简单, 谁都不要玩虚的, 满足**物质需求**是任何事情的前提.  
3. 你担心的事情总是会发生(某理论, 说的虽然绝对, 但有其道理), 我现在担心继续下去, 以后不会有好的生活. 所以我一定会 jump out!
