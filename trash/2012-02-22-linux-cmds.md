---
title: linux 常用命令备忘
layout: post
tags: shell commands
category: linux
---

日常工作中, 这些命令帮我们快速解决问题. 这些命令多是 [GNU toolchain](http://en.wikipedia.org/wiki/GNU_toolchain) 下的 [binutils](http://en.wikipedia.org/wiki/GNU_Binutils) 和 [coreutils](http://en.wikipedia.org/wiki/Coreutils) 里面的.

<!-- ************************************************* -->
#文件
<!-- ************************************************* -->
 
统计文件夹的大小  
 	du -h --max-depth=1 // human-readable, 指定显示文件夹深度
	du -sm // -s: summerize, -m: 按 M byte 单位显示

将程序添加进系统路径  
 	export PATH=$PATH:/path/to/dir1:/path/to/dir2 // per-session
 	vim /etc/profile, 修改 PATH, source(.) /etc/profile // 对所有用户生效

比较多个文件的内容是否一致<sup>[us](http://unix.stackexchange.com/questions/33638/diff-several-files-true-if-all-not-equal)</sup>
    # md5sum * | awk 'NR>1 && $1!=last {exit 100} {last=$1}'

文件描述符 fd 相关([1](http://www.cyberciti.biz/tips/linux-procfs-file-descriptors.html), [2](http://www.cyberciti.biz/faq/linux-increase-the-maximum-number-of-open-files/))

    // 获取进程 id
    # ps aux | grep PROG
    # pgrep PROG
    # pidof PROG
    
    // 获取指定 PID 打开的文件
    # lsof -p PID
    # ls /proc/PID/fd
    
    // 计算所有 fd 数目
    # lsof | wc -l
    
    // 列出 #fd in Kernel Memory
    # sysctl fs.file-nr
    fs.file-nr = 640    0    398358
    含义:
    640    # of allocated fd.
    0      # of unused-but-allocated fd.
    398358 system-wide maximum # of fd.

    // 列出容许 open fd 的最大值
    # cat /proc/sys/fs/file-max
    398358 - normal user can have open in single login session.
    
    // 查看 open files hard/soft limits
    // [ulimit](http://ss64.com/bash/ulimit.html) 是 bash built-in command like cd.

    // http://stackoverflow.com/questions/34588/how-do-i-change-the-number-of-open-files-limit-in-linux
    // soft limits are simply the currently enforced limits
    // hard limits mark the maximum value which cannot be exceeded by setting a soft limit
    # ulimit -Hn 
    # ulimit -Sn

[对 svn workcopy 下的所有文件(除.svn) 外应用 dos2unix](http://stackoverflow.com/a/2228175/264035)

    $ find . -type f \! -path \*/\.svn/\* -exec dos2unix {} \;

<!-- ************************************************* -->
#进程
<!-- ************************************************* -->

[fork bomb](http://www.cyberciti.biz/faq/understanding-bash-fork-bomb/)

    :({ :|:& };:)

[ps aux 输出的含义](http://superuser.com/questions/117913/ps-aux-output-meaning)

    # pa aux 
    USER       PID  %CPU %MEM  VSZ RSS     TTY   STAT START   TIME COMMAND
    timothy  29217  0.0  0.0 11916 4560 pts/21   S+   08:15   0:00 pine  
    root     29505  0.0  0.0 38196 2728 ?        Ss   Mar07   0:00 sshd: can [priv]   
    can      29529  0.0  0.0 38332 1904 ?        S    Mar07   0:00 sshd: can@notty  
    
    USER = user owning the process
    PID = process ID of the process
    %CPU = It is the CPU time used divided by the time the process has been running.
    %MEM = ratio of the process’s resident set size to the physical memory on the machine
    VSZ = virtual memory usage of entire process
    RSS = resident set size, the non-swapped physical memory that a task has used
    TTY = controlling tty (terminal)
    STAT = multi-character process state
    START = starting time or date of the process
    TIME = cumulative CPU time
    COMMAND = command with all its arguments

<!-- ************************************************* -->
#miscellaneous
<!-- ************************************************* -->
 	
查看 dns server
    cat /etc/resolv.conf
    
查看使用的 io scheduler
    # cat /var/log/boot.msg | grep 'io scheduler'
    // 示例输出:
    <6>[    1.393938] io scheduler noop registered
    <6>[    1.394072] io scheduler anticipatory registered
    <6>[    1.394208] io scheduler deadline registered
    <6>[    1.394375] io scheduler cfq registered (default)

将程序放在后台执行, 并且抑制其输出到控制台
	# command > /dev/null 2>&1 &

查看上一个程序执行的状态(通常0代表成功,其他代表失败.查询依据应是errno,这是全局唯一的)
    # echo $?

设置 su 的密码
    # passwd root

修改用户名（非当前登录）
    # usermod -l NEW_NAME OLD_NAME

增加".."和"..."语义
    # vim ~/.bashrc
    // 增加如下内容
    alias ..='cd ..'
    alias ...='cd ../..'

获取page size([ref](http://wangcong.org/blog/archives/764))
    # grep ^KernelPageSize /proc/self/smaps

在 `tail -f` 使用高亮
    # tail -f /var/log/logfile | perl -p -e 's/(something)/\033[7;1m$1\033[0m/g;'
    # (更多: [`tail -f` with highlighting](http://blog.nominet.org.uk/tech/2005/05/26/tail-f-with-highlighting/))

[修改 hostname](http://www.ducea.com/2006/08/07/how-to-change-the-hostname-of-a-linux-system/)

    每个发行版的处理方式还不一样, 这里给出貌似必杀技
    # sysctl kernel.hostname
    # sysctl kernel.hostname=xmachine

