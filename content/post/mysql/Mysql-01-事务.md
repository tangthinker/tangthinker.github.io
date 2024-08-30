---
title: "MySQL 01 事务"
subtitle: ""
date: 2024-08-09T14:15:48+08:00
lastmod: 2024-08-09T14:15:48+08:00
draft: false
keywords: ["mysql", "acid"]
description: "mysql事务"
tags: []

categories: ["mysql"]
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

# 数据库事务

## 事务的概念

事务（Transaction）是指作为一个整体，对一组数据库操作进行集合，要么都成功，要么都失败。事务具有4个属性：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）。

- 原子性（Atomicity）：事务是一个不可分割的工作单位，事务中包括的诸操作要么都做，要么都不做。
- 一致性（Consistency）：事务必须是使数据库从一个一致性状态变到另一个一致性状态。一致性与原子性是密切相关的。
- 隔离性（Isolation）：一个事务的执行不能被其他事务干扰。即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰。
- 持久性（Durability）：持续性也称永久性（Permanence），指一个事务一旦提交，它对数据库中数据的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何影响。


## 事务的实现

事务的实现主要有两种方式：

- 自动提交（auto-commit）：默认情况下，如果没有显式开启事务，则每个查询都被当做一个事务自动提交。
- 事务块（transaction block）：通过 BEGIN、COMMIT、ROLLBACK 语句来显式开启和结束事务。

## 事务的隔离级别

事务隔离级别（Transaction Isolation Level）是指一个事务对另一个事务的影响范围。

- 读未提交（Read Uncommitted）：一个事务可以读取另一个事务未提交的数据。
- 读提交（Read Committed）：一个事务只能读取另一个事务已经提交的数据。
- 可重复读（Repeatable Read）：一个事务在同一个事务中多次读取同一数据时，会看到同样的数据行。
- 串行化（Serializable）：最高的隔离级别，一个事务在同一时间只能执行一个语句，其他语句必须等前一个语句执行完成后才能执行。

## 事务的使用

事务的使用场景：

- 数据库操作（如增删改查）
- 银行转账
- 多用户同时操作同一数据（如多人同时在线编辑文档）


# MySQL 事务

## 实现方式

1. 回滚日志（undo log）：记录所有事务的修改操作，如果发生错误，可以根据回滚日志进行回滚操作。 所有的修改会先记录到回滚日志中，然后再对数据库进行写入。
2. 重做日志（redo log）：记录所有事务的提交操作，用于保证事务的持久性。

### 原子性 - Undo Log

InnoDB实现回滚，靠的是undo log： 当事务对数据库进行修改时，InnoDB会生成对应的undo log，当事务执行失败或调用了rollback，便利用undo log进行回滚。

undo log属于逻辑日志，它记录的是sql的执行信息，当发生回滚的时候，InnoDB会根据undo log进行相反的操作：对于每个insert操作，会执行对应delete操作。对于每个update操作，会执行相反的update操作。

### 持久性 - Redo Log

InnoDB作为存储引擎，数据存放在磁盘中，但是如果每次读写都需要磁盘IO，效率会很低。因此InnoDB提供了Buffer Pool，它是内存中的一个缓存。Buffer Pool包含了磁盘中部分数据页的映射。

当从数据库读取数据时，会首先从Buffer Pool中查找，如果没有，则从磁盘读取，并放入Buffer Pool中。当向数据库写入数据时，会先写入Buffer Pool。**Buffer Pool中的数据会定期刷新到磁盘中**。 这个过程叫做**刷脏**。

但是，如果MySQL发生了宕机，存储在内存中的Buffer Pool的被修改的数据还没有刷新到磁盘中，就会导致数据丢失的问题，无法保证事务的持久性。

于是redo log被引入来解决这个问题：当数据被修改时，使用WAL（Write-Ahead Logging 预写式日志）先记录redo log，然后再更新到Buffer Pool中。当发生crash时，可以利用redo log进行恢复。保证了持久性的要求。

为什么不直接写入磁盘，而要首先写入redo log来解决这个问题？

1. 刷脏是随机IO，因为每次修改数据的位置是随机的。而redo log是顺序IO，属于顺序IO，写入性能更高。
2. 刷脏是以数据页（page）为单位，MySQL默认页面大小为16KB，一个page的小修改都要整页写入。而redo log只包含真正需要写入的部分，无效IO大大减少。


#### redo log与binlog的区别

1. 功能不同：redo log用于保障crash recovery，保证MySQL宕机也不影响持久性。binlog用于point-in-time recovery，保证服务器可以基于时间点进行数据恢复，还可以用于主从复制。
2. 层次不同：redo log是InnoDB引擎层面的，binlog是MySQL服务器层面的。
3. 内容不同：redo log是物理日志，内容基于磁盘的page；binlog的内容是二进制的，根据binlog_format参数的不同，可能基于sql语句、基于数据本身或者二者的混合。
4. 写入时机不同： binlog在事务提交时写入，redo log写入时机：a. 事务提交时调用fsync对redo log进行刷盘； b. master thread每隔一段时间将redo log写入磁盘。


### 隔离性

#### 定义

**与原子性、持久性侧重于研究事务本身不同，隔离性研究的是不同事务之间的相互影响。** 

隔离性指的是： 事务内部的操作与其他事务是隔离的，并发执行的各个事务之间不能相互影响。

严格的隔离性，对应了事务隔离级别中的Serializable（可串行化），但是实际应用是会处于性能方面的考虑而很少使用串行化。

隔离性最求的是并发场景下事务之间的互不干扰。以写操作和读操作来说：

1. 一个事务的写操作对另一个事务写操作对影响：使用**锁机制**来保证隔离性。
2. 一个事务的写操作对另一个事务读操作对影响：使用**MVCC（多版本并发控制）**来保证隔离性。


#### 锁机制

隔离性要求同一时刻只能有一个事务对数据进行写操作。

基本原理：**事务在修改数据之前，需要现货得相应的锁；获取到锁之后，事务才可以修改数据**；事务操作期间，这部分数据是锁定的，其他事务如果需要修改数据，需要等待事务提交或回滚后释放锁。

##### 行锁与表锁

InnoDB支持两种锁：行锁（row-level locking）和表锁（table-level locking）。

表锁在操作数据时会锁定整张表，并发性能较差；行锁只需要锁定需要操作对数据，并发性能较好。

但是在需要锁定的数据较多时，使用表锁可以节省大量开销（获得锁、检查锁、释放锁等）。

**出于性能考虑、绝大多数情况下使用的都是行锁。**

可以通过以下方式查看InnoDB中锁的情况：

```SQL

#锁的概况

select * from information_schema.innodb_locks; 

#InnoDB整体状态，其中包括锁的情况

show engine innodb status; 

```

上面介绍的是排他锁（写锁）之外，MySQL还有共享锁（读锁）。

#### 脏读、不可重复读、幻读 And 事务隔离级别（读未提交、读已提交、可重复读、串行化）

1. 脏读（Dirty Read）：一个事务读到了另一个事务未提交的数据。
    ![脏读](/img/mysql/dirty-read.png)
2. 不可重复读（Non-Repeatable Read）：一个事务在同一个事务中多次读取同一数据时，会看到同样的数据行，但是数据被其他事务修改了。
    ![不可重复读](/img/mysql/non-repeatable-read.png)
3. 幻读（Phantom Read）：一个事务在同一个事务中多次读取同一数据时，**会看到不一样的数据行，因为其他事务插入了新的行。**
    ![幻读](/img/mysql/phantom-read.png)


事务隔离级别：

![事务隔离级别](/img/mysql/isolation-level.png)

在实际应用中，**读未提交**在并发时会出现很多问题，但是**性能相对于其他隔离级别提高的又很有限**，所以使用较少。

**可串行化**强制事务串行，并发效率很低，**只有对数据一致性要求极高时并且可以介绍没有并发时**，才使用。

大多数数据库系统中，默认隔离界别时**读已提交**(Oracle)或**可重复读**(InnoDB)。


#### MVCC 多版本并发控制

Repeated Read（可重复读）解决脏读、不可重复读、幻读，是通过MVCC来实现的。


MVCC（Multi-Version Concurrency Control）是InnoDB存储引擎的一种事务隔离级别，通过保存数据在某个时间点的快照来实现。

下面这个例子体现了MVCC对特点：在同一时刻，不同事务读取到的数据可能是不同的。

![MVCC](/img/mysql/mvcc.png)

MVCC的优势是读取不加锁，因此读写不冲突，并发性能好。InnoDB实现MVCC，多个版本的数据可以共存，主要基于以下技术和数据结构：

1. 隐藏列：InnoDB中每行数据都有隐藏列，**隐藏列中包含了本行数据的事务id、指向undo log的指针等。**
2. 基于undo log的版本链：隐藏列中包含的指向undo log指针，**而每条undo log也会指向更早版本的undo log，从而形成版本链。**
3. ReadView：通过隐藏列和版本链，MySQL可以讲数据恢复到指定的奔波；具体要恢复到哪个版本，则需要根据ReadView来确定。所谓ReadView，是指事务在某一时刻给整个事务系统（trx_sys）打的快照，之后进行读取操作，会讲读取到的数据中的事务id与trx_sys快照进行比较，从而半段数据对该ReadView是否可见，即对该事务可见。

trx_sys中主要内容、判断可见性的方法：

1. low_limit_id：生成ReadView时系统中应该分配给下一个事务的id，如果数据的事务id大于等于low_limit_id，则对该ReadView不可见。
2. up_limit_id：生成ReadView时系统中活跃的读写事务中最小的事务id，如果数据的事务id小于up_limit_id，则对该ReadView可见。
3. rw_trx_ids: 生成ReadView时当前系统中活跃的读写事务的事务id列表。如果事务id在low_limit_id和up_limit_id之间，则需要判断事物id是否在rw_trx_ids中，如果在，说明生成ReadView时事务仍然活跃，因此数据对该ReadView不可见；如果不在，说明生成ReadView时事务已经提交了，因此对该ReadView可见。


![MVCC流程图](/img/mysql/tx_sys.png)


### 一致性

一致性指的是事务执行结束后，数据库的完整性约束没有被破坏，事务执行前后都是合法时数据状态。

前文提到的原子性、持久性和隔离性都是为了保证数据库状态的一致性。

除了数据库层面的保障，一致性还需要应用层面的保障。


# 参考文章

[深入学习MySQL事务：ACID特性的实现原理](https://www.cnblogs.com/kismetv/p/10331633.html
)

