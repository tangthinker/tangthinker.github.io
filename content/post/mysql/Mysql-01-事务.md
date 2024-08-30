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

1. 一个事务的写操作对另一个事务写操作对影响：使用锁机制来保证隔离性。
2. 一个事务的写操作对另一个事务读操作对影响：使用MVCC（多版本并发控制）来保证隔离性。


#### 锁机制

隔离性要求同一时刻只能有一个事务对数据进行写操作。

基本原理：**事务在修改数据之前，需要现货得相应的锁；获取到锁之后，事务才可以修改数据**；事务操作期间，这部分数据是锁定的，其他事务如果需要修改数据，需要等待事务提交或回滚后释放锁。

##### 行锁与表锁

InnoDB支持两种锁：行锁（row-level locking）和表锁（table-level locking）。

表锁在操作数据时会锁定整张表，并发性能较差；行锁只需要锁定需要操作对数据，并发性能较好。

但是在需要锁定的数据较多时，使用表锁可以节省大量开销（获得锁、检查锁、释放锁等）。

**出于性能考虑、绝大多数情况下使用的都是行锁。**
