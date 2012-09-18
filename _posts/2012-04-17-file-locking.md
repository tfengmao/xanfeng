---
title: 文件锁(file locking)
layout: post
tags: file lock mandatory advisory
category: linux
---

[file locking](http://en.wikipedia.org/wiki/File_locking) 是指对一个文件加锁, 或者对文件的某些部分加锁.

util-linux 包里面有 flock 工具, `man 1 flock` 可以看到工具用法, `man 2 flock` 可以看到 API 定义.  
flock 提供 advisory lock 语义, 它的用法是:

    # flock -xn test.lock -c 'vim test.file' // console 1
    # flock -xn test.lock -c 'cat test.file' // console 2, 读取失败, 马上返回, 因为指定 nonblock

python 里面的 [fcntl](http://docs.python.org/library/fcntl.html#module-fcntl) 提供了 fcntl.flock 和 fcntl.fcntl, 它们提供的也是 advisory lock 的语义, 用法是:

    // console 1
    >>> f = open("test.file", "aw")
    >>> fcntl.flock(f.fileno(), fcntl.LOCK_EX)
    >>> // 成功, 开始执行其他操作
    //
    // console 2
    >>> f2 = open("test.file", "aw")
    >>> fcntl.flock(f2.fileno(), fcntl.LOCK_EX) 
    >>> flock 失败, 等待中...

fcntl.fcntl 功能比 fcntl.flock 更全面而强大, 后者就是用前者来实现的.

上面的锁都是建议锁(advisory lock)方式, 需要使用者协调. 不过, [某些 OS 还是支持强制锁(mandatory lock)的](http://kernel.org/doc/Documentation/filesystems/mandatory-locking.txt), 一般是通过设置 FS 的 mount options + set group-id bit + unset group-execute bit 来实现的, 具体可 google 之.
