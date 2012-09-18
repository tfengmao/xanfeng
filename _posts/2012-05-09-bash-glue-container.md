---
title: bash scripts - glue & container
layout: post
tags: glue container script bash temp
category: scripts
---

Linux/Unix 的哲学是: KISS. -- *没错, 就是一定要先找一个恋人, 然后 KISS...*  
所以你会遇到多个**小而精**程序, 比如 /bin 下面.  
我们自己写程序的时候, 也强烈推荐参考这种思路.

但总有一些任务, 涉及多个小程序的多个功能. 此时, 我们可以通过 shell(一般是 bash) scripts 去胶合(glue)这些小功能, 或者试想象一下: shell script 是一个容器(container), 我们把小程序扔进去, 放到固定的位置(就像拼图), 然后得到一个"大程序".

下面是一个示例, 用到的思路和技巧:  
1. 用 mktemp 创建临时文件([ref](http://www.cyberciti.biz/tips/shell-scripting-bash-how-to-create-temporary-random-file-name.html))  
2. 重定向 stderr/stdout 到临时文件.  
3. split string.  

<script src="https://gist.github.com/2643359.js"> </script>