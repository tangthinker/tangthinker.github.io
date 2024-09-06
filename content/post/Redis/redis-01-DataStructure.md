---
title: "Redis Data Structure"
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
toc: true
autoCollapseToc: true
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

Redis中广泛应用字典/Map，主要有以下两个用途：

1. 实现数据库键空间（ key space ）。
2. 用作Hash类型键的底层实现之一。

Hash类型使用字典和ziplist作为底层实现，因为压缩列表比字典占用内存更少，所以在创建新的Hash键时，会优先使用ziplist作为底层实现，当有需要时，才会使用字典作为底层实现。

字典实现方式有多种：

1. 简单实用链表或数组实现，适用于元素个数不多的情况。
2. 兼顾高效和简单性，可使用哈希表实现字典。
3. 如追求更为稳定的性能特征，并能高效的实现排序操作，可使用更为复杂的平衡树实现。

字典的结构定义如下：
```c

// 每个字典使用两个哈希表，用于实现渐进式rehash
typedef struct dict {
    dictType *type;         // 特定于类型的处理函数
    void *privdata;         // 特定于类型的处理函数私有数据
    dictht ht[2];           // 哈希表（ 2个 ）
    long rehashidx;         // 记录rehash进度的标志，值为-1 表示rehash未进行
    unsigned long iterators;  // 当前正在运行的安全迭代器数量
} dict;

```

字典所使用的哈希表实现类型的定义如下：
```c
typedef struct dictht {
    dictEntry **table;       // 哈希表节点指针数组（ 俗称桶 bucket）数组每个元素指向一个dictEntry结构
    size_t size;             // 指针数组的大小
    unsigned long size;      // 指针数组的大小
    unsigned long sizemask;  // 指针数组的长度掩码，用于计算索引值
    unsigned long used;      // 哈希表现有的节点数量
} dictht;
```

哈希表节点（ dictEntry ）结构定义如下：
```c
typedef struct dictEntry {
    void *key;              // 键
    union {
        void *val;          // 值
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;  // 指向下一个哈希表节点，形成链表
} dictEntry;
```

next指针指向另一个dictEntry结构，从而形成链表，可以看出，dictht使用**链地址法解决哈希冲突**。

当多个不同的键拥有相同的哈希值时，哈希表使用一个链表将这些键连接起来。

字典的哈希表实现如下图所示：

![字典哈希表实现](/img/redis/01/dict-hash.png)


字典总结：

1. Redis中数据库键空间和哈希键类型都是基于字典来实现的。
2. Redis字典的底层实现为哈希表，每个字典使用两个哈希表，一般情况只使用0号哈希表，rehash的过程中才会同时使用两个哈希表。
3. Redis字典的哈希表使用链地址法解决哈希冲突。
4. Rehash可以用于扩展和收缩哈希表。
5. 对哈希表的rehash是分多次、渐进式的进行的。


### 跳表

Redis的有序集合是使用跳表来实现的。

维基百科中跳表示意图：
![跳表](/img/redis/01/skiplist-wiki.png)

可以看出跳跃表主要由以下部分构成：

1. 表头（head）：负责维护跳跃表的节点指针。
2. 层：保存着指向其他元素的指针。高层的指针越过的元素数量大于等于底层指针，为保证查找效率，程序总是从高层开始访问，然后缩小元素值范围，慢慢降低层次。
3. 表尾：全部由NULL节点构成，表示跳跃表的尾部。
4. 节点：保存着元素值，以及多个层。

跳表的结构如下：

```c

typedef struct zskiplist {
    // 头节点   尾节点
    struct zskiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 目前表中层数最大的节点的层数
    int level;
} zskiplist;

typedef struct zskiplistNode {
    // 元素value
    sds ele;
    // 元素score
    double score;
    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 这个层跨越的节点数量
        unsigned long span;
    } level[];
} zskiplistNode;
```

跳表按照以上结构创建的示意图：
![跳表](/img/redis/01/skiplist-real.png)

跳表的特点如下：

1. 跳表是一种有序的数据结构，支持快速的查找、插入和删除操作。
2. 跳表的平均时间复杂度为O(logN)，最坏时间复杂度为O(N) - 逐个遍历。
3. 每个节点带有高度为1层的后退指针，用于从表尾向表头遍历。

实际上跳表就是在数组的基础上，**增加多级索引**，从而提高查找效率。

![跳表](/img/redis/01/skiplist.png)


### ziplist 压缩列表

Redis的列表、哈希表、有序集合在数据量较小的时候，会使用压缩列表来存储。

ziplist由一系列特殊编码的内存块构成的列表，每个节点可以保存一个长度受限的字符数组或者整数。

ziplist典型的分布结构：

``` 
area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                         ZIPLIST_ENTRY_TAIL
```

其中：

- zlbytes：记录整个压缩列表占用的内存字节数。在ziplist内存重新分配和计算末端时使用。
- zltail：到达ziplist尾部的偏移量，通过这个偏移量，可在不遍历整个ziplist的条件下快速定位到尾节点。
- zllen：ziplist中节点的数量。当ziplist节点数量小于UINT16_MAX时，这个值就是节点数量；当这个值是UINT16_MAX，需要遍历整个ziplist才能获取节点数量。
- entryX：压缩列表的各个节点，节点的长度由节点保存的内容决定。
- zlend：特殊值0xFF（十进制255 二进制1111 1111），用于标记压缩列表的末端。


压缩单个节点的结构如下：

```c
typedef struct zlentry {
    // 保存前一节点的长度所需的长度
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    // 前一节点的长度
    unsigned int prevrawlen;     /* Previous entry len. */
    // 保存节点的长度所需要的长度
    unsigned int lensize;        /* Bytes used to encode this entry len. */
    // 节点长度
    unsigned int len;            /* This entry len. */
    // header长度
    unsigned int headersize;     /* prevrawlensize + lensize. */
    // 编码方式
    unsigned char encoding;      /* Entry encoding (4 bits). */
    // 节点内容
    unsigned char *p;            /* Pointer to the entry. */
} zlentry;
```

压缩列表的特点如下：

1. 压缩列表可以节省内存空间。
2. 添加和删除ziplist节点可以会引起连锁更新，因而最坏时间复杂度为O(N^2)，不过出现概率不是很高，可以将添加和删除视为O(N)。