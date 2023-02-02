---
layout:     post
title:      go对map的实现与java对比
subtitle:   go对map的实现给出了与java截然不同的解决方案，还是比较有意思的。
date:       2023-2-2
author:     亚热带小番茄
header-img: img/blog/default_header.png
catalog: true
tags:
- go基础
---

> 引言：go对map的实现给出了与java截然不同的解决方案，还是比较有意思的。当java中的map遇到哈希冲突使用链地址法，而go使用典型的开放寻址法。在对 sync map的实现，go的方法很好体现了用空间换时间的思想。

# go map原理

和java的map差不多，java在发生哈希冲突时使用链地址法，而go是典型的开放寻址法，使用的线性探测。

![](img/blog/2023-2-2-goMap.png)

go map同样是有桶，一个桶内存放的一个长度为8的数组、一个tophash数组用于保存hashcode的高位、一个overflow当这个数组不够存了用于指向下一个数组。

当插入数据时，先取hashcode，然后取模确定是哪一个桶。然后在数组中对比hash的高位，如果同样再对比全部的hashcode，一样就替换， 不一样则向下线性寻找第一个为空的位置插入。如果数组满了，那就创建一个新的数组，将overflow指向他。get同理。

这样当数据量比较大看似比较慢，get时间复杂度达不到1。为了保证查询速度，go map扩容，防止数据量大时overflow太长。当 map中元素/桶数 > 6.5（每个桶元素平均个数大于6.5）时；或overflow的bucket太多时，发生扩容。扩容是扩容一倍，然后采用申请新的空间，把老数据复制进去。

> go map 与 java map对比

1. go map的机制导致需要提前申请内存空间，用不掉的话可能比较浪费

2. java就不太会浪费，但是java需要存储指针(链表 或 红黑树)

3. 当数据量比较小时，两个get的操作速度差不多，但是数据量大时java采用红黑树，get的速度会快一些。但是set 和delete都大不如go，因为还需要保证红黑树平衡。

# sync.map

java为了线程安全又兼顾效率，采用分段锁，锁的粒度是桶。go采用了空间换时间的思想，加快速度。

sync.map 中又有个read、dirty。read主要用来读，dirty用来写。读read是不用加锁的，但是读写dirty需要加锁。当read读不到时再去读dirty。为了防止读dirty太多影响性能，有一个miss字段来统计读dirty次数，达到了一定的次数dirty会想read同步数据。这个步骤是加锁的。

写时，先会从read中查询是否有这个数据，如果有的话，拿到这一块数据的内存指针，cas这一块内存尝试更新。如果不在则上锁，更新dirty里面的数据。删除也是一样

go 锁的粒度是整个dirty，但是有一个read挡在他前面，相当于有一个缓存。可以在读写场景不冲突的情况下可以不用加锁，提升速度。

可以看出 go的map适合读多写少的场景。
