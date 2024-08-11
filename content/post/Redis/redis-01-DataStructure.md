---
title: "Redis 01 Data Structure"
date: 2024-08-09T14:15:48+08:00
lastmod: 2024-08-09T14:15:48+08:00
draft: false
keywords: ["redis", "data structure"]
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


## Redis底层数据结构

### SDS 简单动态字符串

Redis的字符串是使用SDS（Simple Dynamic String）来实现的。

SDS的结构如下：

```c
struct sdshdr {
    int len; // 记录buf数组中已使用字节的数量，等于SDS所保存字符串的长度
    int free; // 记录buf数组中未使用字节的数量
    char buf[]; // 字节数组，用于保存字符串
};
```

SDS与C语言中的字符串相比，有以下优点：

1. 获取字符串长度的时间复杂度为O(1)，而C语言中的字符串需要遍历整个字符串。
2. SDS可以自动扩容，而C语言中的字符串需要手动分配内存。
3. SDS可以存储二进制数据，而C语言中的字符串只能存储文本数据。



### 双端链表

Redis的列表是使用双端链表来实现的。

Redis中列表使用两种数据结构作为底层实现：ziplist和linkedlist。

因为双端列表占用内存比压缩列表多，所以在创建新的列表键是，列表会优先使用ziplist作为底层实现，当元素个数超过512个，或者单个元素超过64字节时，会使用linkedlist作为底层实现。

双端链表的结构如下：

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
    // value类型为void*，可以存储任意类型的数据
} listNode;

typedef struct list {
    listNode *head;                 // 表头指针
    listNode *tail;                 // 表尾指针
    unsigned long len;              // 节点数量
    void *(*dup)(void *ptr);        // 复制节点值函数
    void (*free)(void *ptr);        // 释放节点值函数    删除节点时调用
    int (*match)(void *ptr, void *key); // 对比节点值和输入值函数
} list;
```

双端链表的特点如下：

1. 节点带有prev和next指针，可以进行双向遍历。
2. List保存了head和tail两个指针，因此对表头和表尾的插入和删除操作复杂度为O(1)。 这是实现高效的**LPUSH、LPOP、RPUSH、RPOP**操作的重要保证。
3. List保存了len属性，因此计算列表长度的时间复杂度为O(1)。这是实现高效的**LLEN**操作的重要保证。
4. List使用dup、free、match三个函数指针，可以用于复制、释放、对比节点值，从而实现多态。

双端列表迭代器

Redis为双端列表实现了一个迭代器，迭代器可以从两个方向对双端列表进行迭代。

迭代器的结构如下：

```c
typedef struct listIter {
    listNode *next; // 下一个节点
    int direction; // 迭代方向
} listIter;
```

- 可以沿着节点的next指针，从表头向表尾迭代。 direction为adlist.h/AL_START_HEAD
- 可以沿着节点的prev指针，从表尾向表头迭代。 direction为adlist.h/AL_START_TAIL



### 字典/Map/哈希表

Redis的字典是使用哈希表来实现的。

哈希表的结构如下：
```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

字典的特点如下：

1. 字典可以快速地查找、插入和删除键值对。
2. 字典可以自动扩容和缩容。
3. 字典可以处理哈希冲突。


### ziplist 压缩列表

Redis的列表和哈希表在数据量较小的时候，会使用压缩列表来存储。

压缩列表的结构如下：

```c
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry len. */
    unsigned int len;            /* This entry len. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Entry encoding (4 bits). */
    unsigned char *p;            /* Pointer to the entry. */
} zlentry;
```

压缩列表的特点如下：

1. 压缩列表可以节省内存空间。
2. 压缩列表可以快速地插入和删除节点。
3. 压缩列表可以快速地获取头节点和尾节点。
4. 压缩列表可以快速地获取链表的长度。


### 跳表

Redis的有序集合是使用跳表来实现的。

跳表的结构如下：

```c
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

跳表的特点如下：


1. 跳表可以快速地查找、插入和删除节点。
2. 跳表可以快速地获取头节点和尾节点。
3. 跳表可以快速地获取链表的长度。
4. 跳表可以处理哈希冲突。

实际上跳表就是在数组的基础上，增加多级索引，从而提高查找效率。

![跳表](/img/redis/01/skiplist.png)