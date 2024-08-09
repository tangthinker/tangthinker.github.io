---
title: "Redis 01 Data Structure"
date: 2024-08-09T14:15:48+08:00
lastmod: 2024-08-09T14:15:48+08:00
draft: false
keywords: ["redis", ""]
description: "Redis数据结构介绍"
tags: []
categories: ["redis"]
author: "Tangthinker"

# Uncomment to pin article to front page
# weight: 1
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false

# Uncomment to add to the homepage's dropdown menu; weight = order of article
# menu:
#   main:
#     parent: "docs"
#     weight: 1
---

<!--more-->

## Redis 五种数据类型

1. String 字符串
2. Hash 哈希
3. List 列表
4. Set 集合
5. Zset 有序集合

### Redis数据数据类型与底层数据结构对应关系

![Redis五种数据类型与底层数据结构对应关系](/img/redis/01/ds.png)

## Redis键值对查找关系

Redis为了实现从key到value到快速访问，使用哈希表的方式来存储所有键值对。

哈希表中的entry保存的并不是键值本身，而是存储的是指向具体值的指针。

使用哈希表的，可以让我们在O(1)时间复杂度下，获取指定key对应的value值。

![Redis键值对查找关系](/img/redis/01/hash.png)

## Redis哈希冲突时rehash的过程

当我们往哈希表中写入大量的数据时，哈希冲突是不可避免的。

Redis为了解决哈希冲突，采用了链式哈希。

当发生哈希冲突时，Redis会将冲突的键值对保存在链表中。

![Redis产生hash冲突](/img/redis/01/hash-conflict.png)

当数据量巨大时，哈希冲突链表上的节点越来越多，而其上的元素只能使用遍历的方式查找，时间复杂度为O(n)。

Redis为了解决哈希冲突导致链表过长的问题，采用了rehash。

实际上Redis默认使用了两个全局哈希表，哈希表1和哈希表2。

rehash过程如下：

1. 为哈希表2分配更大的空间，例如是当前哈希表大小的两倍。
2. 将哈希表1中的所有键值对重新计算哈希值，并插入到哈希表2中。
3. 当所有键值对都迁移完毕后，释放哈希表1，将哈希表2设置为哈希表1，然后哈希表2初始化为空表。

在这个过程中，会涉及到大量的数据拷贝，会造成Redis线程阻塞，无法完成其他服务请求。

## Redis渐进式rehash

Redis为了解决rehash过程中线程阻塞的问题，采用了渐进式rehash。

渐进式rehash是指，在rehash的过程中，每次访问哈希表时，都会将哈希表1中的部分键值对迁移到哈希表2中。

这样，在rehash的过程中，哈希表1和哈希表2会同时存在，哈希表1中的键值对会逐渐迁移到哈希表2中。

![Redis渐进式rehash](/img/redis/01/rehash.png)
