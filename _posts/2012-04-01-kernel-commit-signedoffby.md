---
title: Linux kernel maillist 中的 signed-off-by
layout: post
tags: linux kernel maillist tags
category: linux
---

愚人节快乐。  

在Linux kernel maillist的邮件头中经常看到各种tag，如signed-off-by，reported-by等。  
这些标签代表什么含义呢？在kernel源码包的Documentation/SubmittingPatches文件中有描述：

    Signed-off-by：表示signer参与了patch的开发或者在patch的delivery path中。

这个文档中还提到：

1. Acked-by:
2. Cc:
3. Reported-by:
4. Tested-by:
5. Reviewed-by:
6. Developer's Certificate of Origin