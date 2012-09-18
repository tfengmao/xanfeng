---
title: diff & patch
layout: post
tags: diff patch
category: linux
---

[Ten minute guide to diff and patch](http://jungels.net/articles/diff-patch-ten-minutes.html)  
[Introduction: Using diff and patch (tutorial)](http://www.linuxtutorialblog.com/post/introduction-using-diff-and-patch-tutorial)  
[Comparing and Merging Files](http://www.gnu.org/software/diffutils/manual/html_mono/diff.html)  

"Comparing and Merging Files" 的笔记:

---

#diff

TIPS:  
- 用 `diff3` 比较 3 个文件的差异, 用 `sdiff` side-by-side 地显示文件差异.  
- "-E" 或 "--ignore-tab-expansion" 忽略 tab 和空格之间的差异.  
- "-b" 或 "--ignore-space-change" 忽略行尾的空格, 并认为连续的空白符都是一样的. 比 "-E" 强.  
- "-w" 或 "--ignore-all-space" 忽略一行中所有的空白差异. 如果一行有空格, 另一行甚至没有空格, 都认为无差异.  
- "-B" 或 "--ignore-blank-lines" 忽略空白行差异.  
- "-i" 或 "--ignore-case" 忽略大小写差异.  
- "-I regexp" 或 "--ignore-matching-lines=regexp" 忽略正则表达式的匹配行.  
- "-q" 或 "--brief" 只显示是否有差别, 不显示具体信息.  


**Hunks**

一组连续的差异行称为 hunks. gnu diff 的原则: 使所有 hunks 的大小最小, 尽量找出更多的匹配行.

**Output Formats**

[Normal Format](http://www.gnu.org/software/diffutils/manual/html_mono/diff.html#Detailed%20Normal): `diff f1, f2`, "l**a**r, f**c**t, r**d**l"

[Context Format](http://www.gnu.org/software/diffutils/manual/html_mono/diff.html#Context%20Format): `diff -C #lines(--context[=#lines], -c, 默认3行) f1, f2`

[Unified Format](http://www.gnu.org/software/diffutils/manual/html_mono/diff.html#Detailed%20Unified): `diff -U #lines(--unified[=#lines], -u, 默认3行) f1, f2`  
目前仅 gnu diff 支持这种格式, 为了 patch works properly, 一般需要至少3行的 context.  

Unified Format 的详细说明:  

> --- from-file from-file-modification-time  
> +++ to-file to-file-modification-time

比较的两个文件.

> @@ from-file-range to-file-range @@  
>  line-from-either-file  
>  line-from-either-file...  

hunk. 由多个这种格式的 hunks 构成这两个文件的 diff. 其中,  
+ 表示在 from(first) file 中加入行  
- 表示在 from(first) file 中移除行

示例:  

> --- lao	2002-02-21 23:30:39.942229878 -0800  
> +++ tzu	2002-02-21 23:30:50.442260588 -0800  
> @@ -1,7 +1,6 @@  -> 表示比较 from file L1 开始的7行和 to file L1 开始的6行.  
> -The Way that can be told of is not the eternal Way;  
> -The name that can be named is not the eternal name.  
>  The Nameless is the origin of Heaven and Earth;  
> -The Named is the mother of all things.  
> +The named is the mother of all things.  
> +  
>  Therefore let there always be non-being,  
>    so we may see their subtlety,  
>  And let there always be being,  
> @@ -9,3 +8,6 @@  -> 表示比较 from file L9 开始的3行和 to file L8 开始的6行.  
>  The two are the same,  
>  But after they are produced,  
>    they have different names.  
> +They both may be called deep and profound.  
> +Deeper and more profound,  
> +The door of all subtleties!  

"-p" 显示区别所在的函数, 更多见[这里](http://www.gnu.org/software/diffutils/manual/html_mono/diff.html#C%20Function%20Headings).  

[side by side format](http://www.gnu.org/software/diffutils/manual/html_mono/diff.html#Side%20by%20Side%20Format): `diff -y(--side-by-side) -W #cols(--width=#cols) f1 f2`

---

#patch

selecting the patch input format:  
-c, --context: context diff.  
-e, --ed: ed script.  
-n, --normal: normal diff.  
-u, --unified: unified diff.

applying patches with changed white space:  
-I, --ignore-white-space: 忽略空白符的差异.

applying reversed patches:  
-R, --reverse: patch 会去交换每一个 hunk 的 from 和 to 部分. 这适合于错误地执行 `diff new old` 产生的 patch 或者要恢复刚刚打上的 patch.

to be continued...

