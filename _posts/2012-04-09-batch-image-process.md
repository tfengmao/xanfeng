---
title: ubuntu 批量压缩照片
layout: post
tags: ubuntu image convert batch
category: desktop
---

缘起:杭州绕钱塘江骑行了一圈,拍了很多照片.想压缩下传到豆瓣.

根据"[Linux下用批量convert管理图片](http://www.linuxbyte.org/linux-convert-mini-howto.html)",下个软件,写个小脚本搞定批量压缩.

    # sudo apt-get install imagemagick
    # vim img.sh
    cd pics_from
    for img in `ls *`
    do
        convert -resize 40%x40% $img sm-$img
    done

`man convert`可以得到 imagemagick 的更多信息.
