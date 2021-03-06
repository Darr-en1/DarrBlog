---
title: mysql-lock
copyright: true
permalink: 1
top: 0
date: 2020-11-08 16:39:54
tags:
    - mysql
    - lock
categories: mysql
password:
---


锁是计算机协调多个进程或线程并发访问某一资源的机制。而根据加锁的范围，可被分为全局锁、表级锁和行锁。他们都有什么作用呢？有让我们一探究竟吧！<!--more-->

## 全局锁

### FTWRL

全局锁将对整个数据库实例加锁，使得整个库处于只读状态，会阻塞DDL和部分DML(select不被阻塞)语句。
通过输入命令 Flush tables with read lock (FTWRL)让整个库处于只读状态。

全局锁的典型使用场景是做全库逻辑备份。
但是在备份过程中整个库完全处于只读状态，因此会出现一些隐患：
- 如果在主库上备份，那么在备份期间都不能执行更新，业务基本上就得停摆；
- 如果你在从库上备份，那么备份期间从库不能执行主库同步过来的binlog，会导致主从延迟。

如此大的隐患势必在大部分业务场景下是不能接受，那有没有更好的办法呢？
### single-transaction 实现数据库备份
InnoDB利用了“所有数据都有多个版本”的这个特性，实现了“秒级创建快照”的能力。


我们可以通过一致性视图实现数据库备份。当我们使用InnoDB时，倘若我们将事务隔离级别设置成可重复读。
事务在启动的时候基于整库进行"快照"。因此我们便可以对当前数据进行备份，并不影响其他事务操作。

mysql官方自带的逻辑备份工具 mysqldump。

当mysqldump使用参数–single-transaction的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于MVCC的支持，这个过程中数据是可以正常更新的。
但这又存在一个问题。它需要数据库存储引擎处于一致性读的事务隔离级别。倘若使用MYISAM，则不能使用该功能。

### readonly 和 FTWRL
mysql可以设置`set global readonly=true`将数据库变成全库只读。但他两的适用范围却不一样：
- 在有些系统中，readonly的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，修改global变量的方式影响面更大。
- 在异常处理机制上有差异。如果执行FTWRL命令之后由于客户端发生异常断开，那么MySQL会自动释放这个全局锁，整个库回到可以正常更新的状态。
而将整个库设置为readonly之后，如果客户端发生异常，则数据库就会一直保持readonly状态，这样会导致整个库长时间处于不可写状态，风险较高。

## 表级锁
MySQL里面表级别的锁有两种：表锁和元数据锁（meta data lock，MDL)。

### 表锁

MyISAM存储引擎只支持表锁。

表锁的语法是 lock tables … read/write。通过 unlock tables主动释放锁，也可以在客户端断开的时候自动释放。

当一个线程获取到表级写锁后，只能由该线程对表进行读写操作，别的线程必须等待该线程释放锁以后才能操作

当一个线程获取到表级读锁后，该线程只能读取数据不能修改数据，其它线程也只能加读锁，不能加写锁

### MDL
MDL在MySQL 5.5版本中引入。MDL不需要显式使用，在访问一个表的时候会被自动加上。

**当对一个表做增删改查操作的时候，加MDL读锁**；

**当要对表做结构变更操作的时候，加MDL写锁**。


读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。

读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

申请MDL锁的操作会形成一个队列，队列中写锁获取优先级高于读锁。一旦出现写锁等待，不但当前操作会被阻塞，同时还会阻塞后续该表的所有操作。事务一旦申请到MDL锁后，直到事务执行完才会将锁释放。

**如果A会话开启一个事务对t表进行了一次查询并没有提交，该操作会获取MDL读锁。然后B会话对t表字段进行更改，上一个事务没有提交MDL读锁没有被释放，因此B会话block。
但是之后所有要在表t上新申请MDL读锁的请求也会被B会话阻塞。**

#### 如何安全地给小表加字段？

首先我们要解决长事务，事务不提交，就会一直占着MDL锁。
在MySQL的information_schema 库的 innodb_trx 表中，你可以查到当前执行中的事务。
如果你要做DDL变更的表刚好有长事务在执行，要考虑先暂停DDL，或者kill掉这个长事务。
倘若要变更的表是一个热点表，虽然数据量不大，但是上面的请求很频繁，这时候kill可能未必管用，因为新的请求马上就来了。
比较理想的机制是，在alter table语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到MDL写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。
之后开发人员或者DBA再通过重试命令重复这个过程。

#### Online DDL
mysql 5.6以前 在DDL执行期间其他DML不能并行，运维人员对表的结构更改会导致该表的业务停滞。
这在很多时候是不能容忍的，因此在mysql5.6 引入了Online DDL ,它使得在DDL执行期间其他DML可以并行。

Online DDL下，当获取到MDL写锁会将其降级成MDL读锁，然后真正做DDL操作。因此在DDL操作的过程中，表处于读锁状态。
DML不被阻塞。DDL操作完成后会升级成MDL写锁然后释放MDL锁。

## 行锁
行锁是粒度最小的锁。在Innodb引擎中既支持行锁也支持表锁，但只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。
行锁开销会增大，加锁速度慢，并会出现死锁。但因为锁粒度变小，发生锁冲突的概率低，处理并发的能力强。

### 种类
**共享锁（S锁）**: 也称读锁，允许其他事物再加S锁，不允许其他事物再加X锁

**排他锁（X锁）**: 也称写锁，不允许其他事务再加S锁或者X锁

### 加锁方式

#### 自动加锁
对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加排他锁；
对于普通SELECT语句，InnoDB不会加任何锁；

#### 手动加锁
普通SELECT语句是不会加锁的，但是也可以手动加锁。

共享锁：select...lock in share mode

排他锁：select ... for update

明明普通SELECT查询数据，读写不冲突塞，为什么还要对其上锁呢？

因为mysql读操作其实分为2种类型:**快照读（snapshot read）**及**当前读（current read）**。
前者基于Mysql mvcc实现,后者是对数据库当前数据块的读取。

UPDATE、INSERT、DELETE 和 加锁的SELECT 都是当前读，而普通的SELECT则是快照读。

### next-key lock

行锁只能锁住行，并不能锁住后面加入满足条件的行，这是幻读产生的原因。
如果能对一个范围进行加锁，保证这个范围内数据不变，就可以消除幻读的问题。
**间隙锁（Gap Lock）** 则是Innodb在可重复读隔离级别下提供的一种锁定范围左闭右闭的区间锁。
Innodb通过行锁和间隙锁共同组成的（next-key lock）从而解决了幻读问题。

可重复读隔离级别下加锁规则：
- 加锁的基本单位是next-key lock（前开后闭区间）。
- 查找过程中访问到的对象才会加锁。
- 索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁。
- 索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock退化为间隙锁。


参考：

[间隙锁详解](https://www.jianshu.com/p/32904ee07e56)
[MySQL实战45讲](https://time.geekbang.org/column/intro/139)

