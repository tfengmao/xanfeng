---
title: Linux 中的 Page Cache
layout: post
tags: linux page-cache direct io
category: linux
---

page_cache（页高速缓存）由RAM中的物理页组成，缓存中每一页都对应磁盘中的多个块。

除了O_DIRECT之外，所有read和write都依赖于page_cache。

page_cache中的数据单位是整页整页的数据，不过一个页面中的数据在磁盘上不必是相邻的，因此页面不能用设备号和块号来识别。page_cache中的页面是通过拥有者和拥有者数据中的索引（通常是inode和相应文件内的偏移量）识别的。

page_cache的核心数据结构是address_space，每个页面描述符（struct page）包含两个成员：i_mapping指向inode（拥有该页面）的address_space对象，index是拥有者“地址空间”（此时可以将地址空间理解为磁盘上的文件）内的偏移量，单位大小是页。

若page_cache中的页的所有者是文件，address_space对象就嵌入在VFS inode对象的i_data字段中。i_mapping字段总是指向含有inode数据的页所有者的address_space对象。

**参考资料**：  
Linux内核设计与实现

*many thanks to the cannot-named hackers.*