---
title: innodb锁类型(翻译自官网5.7版本)
date: 2018-08-31 17:34:05
toc: true
tag:
	- mysql
	- innodb
	- lock
categories:
    - 阅读
---

## 共享锁与排它锁

InnoDB实现了标准行级锁，有两种类型共享锁（S锁）和排他锁（X锁）

- 一个共享锁（S锁）允许持该锁的事务读取一行数据

- 一个排他锁（X锁）允许持该锁的事务更新或者删除一行数据

如果事务T1持有第 r 行的S锁，接着从不同的事务T2发起锁请求被处理有以下情况：

- T2请求S锁会被立即授予，结果是在第r行上，T1和T2都持有第r行的S锁

- T2请求X锁不会被立即授予。

如果事务T1持有第 r 行的X锁，从不同的事务T2发起对 r 行任意锁类型的请求不会被立即授予。相反，事务T2不得不等待T1释放掉它持有在第 r
<!--more-->
行的锁 （即X锁，译者注）

## 意向锁

InnoDB支持多粒度锁，这就允许行锁和表锁共存。比如像 LOCK TABLES ... WRITE语句会在持有一个排它锁(X锁)在指定的表上，为了在多粒度级别实际可用,InnoDB使用意向锁。意向锁是表级锁，

这表明在一个事务在稍后会请求一个行级锁（共享锁或排它锁），这里有两种类型的意向锁

- 一个意向共享锁(IS锁)表明一个事务打算在表中的特定行设置一个共享锁

- 一个意向排它锁(IX锁)表明一个事务打算在表中的特定行设置一个排它锁

比如，SELECT ... LOCK IN SHARE MODE 放置一个IS锁，SELECT ... FOR UPDATE放置一个IX锁

意向锁协议如下：

- 在一个事务能取得表中的某一行共享锁之前，他必须先取得该表上的IS锁或者更高级别的锁。

- 在一个事务能取得表中的某一行排它锁之前，他必须先取得该表上的IX锁

表级锁类型兼容性在下面的矩阵总结：

|     | X    | IX   | S		 | IS	 |
|---- | ---- | ---- | ---- | ---- |
| X   | 冲突  | 冲突 | 冲突	 | 冲突 |
| IX  | 冲突  | 兼容 | 冲突   | 兼容 |
| S   | 冲突  | 冲突 | 兼容	 | 兼容 |
| IS  | 冲突  | 兼容 | 兼容	  | 兼容 |


如果一个事务的请求锁和已存在的锁兼容，则会被授予。如果与已存在锁冲突则不会被授予。事务将会等待直到造成冲突的已存在锁已经释放，如果一个锁请求与现有锁造成了冲突，那么不会被授予。因为这会造成死锁，引发错误。

意向锁除对整张表的请求（比如，LOCK TABLES ... WRITE）外，不会阻塞任何操作。意向锁的主要作用是说明一些人锁了表中一行数据或者准备锁表中一行数据。

对于一个意向锁的事务数据展示和通过SHOW ENGINE INNODB STATUS语句中是相似的，innoDB监视器输出如下：

```mysql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

## 行锁(record lock)

一个行锁锁住一条索引记录，比如SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;防止其他事务插入，更新，或者删除t.c1为10的该行数据。

行锁总是锁住索引记录，即使一个表定义时没有索引，对于这样的情况，InnoDB创建一个隐藏聚合索引，然后用这个在行锁时使用这个索引，参见Section 14.8.2.1, “Clustered and Secondary Indexes”.

对于一个行锁的事务数据展示和通过SHOW ENGINE INNODB STATUS语句中是相似的，innoDB监视器输出如下：

```mysql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## 间隔锁

一个间隔锁锁住两条索引记录的间隙，或者锁住第一条索引记录前或最后一条索引记录后的间隙，例如SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE，防止其他事务插入t.c1值为15的数据，无论在这些列中是否已经有该数据，因为在这个范围内的所有存在数据的间隔已经被所定。

一个间隔可能跨越单个索引，多个索引，甚至是没有索引。

间隔锁是在性能和并发之间权衡的做法，在一些事务隔离级别中使用，一些中不使用。

在使用唯一索引去查找唯一行记录的锁行中，间隔锁是不需要的。（这不包括这样的情况，查询条件仅包括一个多列唯一索引中的一部分列，在这样的情况中，间隔锁产生）。例如，如果id列有唯一索引，下面的语句对于id为100的行将仅仅使用一个索引-记录锁，其他回话是否在之前的间隙插入行无关紧要。

```mysql
SELECT * FROM child WHERE id = 100;
```

如果id不是索引或者没有唯一索引，这条语句会锁住之前的间隙。

这儿值得注意的是，不同的事务可以持有一个间隔中的冲突锁，例如事务A可以持有一个共享间隙锁(间隙S锁)在一个间隙中，以此同时，事务B持有一个排他间隙锁(间隙X锁)在同样的间隙中，冲突间隙锁被允许的原因是如果一个记录从索引上被删掉，必须合并不同事务在记录上持有的间隙锁

InnoDB中的间隙锁是”纯粹禁止“，这意味着他们唯一目的是防止其他事务在间隙中进入插入操作。间隙锁可以共存，一个事务持有的间隙锁不会阻止其他事物取得相同间隔的间隙锁，在共享间隙锁和排他间隙锁之间没有区别，他们互相之间不会冲突，他们是执行相同的功能。

间隙锁可以被明确禁止，如果你将事务隔离级别改为READ COMMITTED或者开启innodb_locks_unsafe_for_binlog系统变量(现在已弃用)，就会被禁止。在这些情况下，间隙锁对于查找和索引扫描被禁止，仅仅在外键约束检查和重复键检查被使用

将事务隔离级别改为READ COMMITTED或者开启innodb_locks_unsafe_for_binlog系统变量还有其他影响，在Mysql评估where条件后，对于不匹配行的行锁会被释放，对于更新语句，InnoDB将进行“半一致”读取，以至于他会返回最新的提交版本给Mysql,让Mysql决定匹配该行是否和update语句的where条件匹配。

## 下一键锁(Next-Key Locks)

下一键锁是索引记录上的行锁和在索引记录前的间隔中的间隙锁的组合

InnoDB通过这样的方式执行行级锁定，当它查找或者扫描一个表索引，会在遇到的索引记录上设置共享锁或者排它锁，这样行级锁实际上是索引记录锁，下一键锁在索引记录上也影响该索引记录之前的间隙，就是说，下一键锁是一个索引记录锁加上在索引记录钱的间隙的间隙锁。如果一个会话持有索引上记录R的共享锁或者排它锁，其他回话在索引记录里在R之前的间隙中不能立即插入一条新的索引记录。

假设一个索引包含以下值，10，11，13，20。对于这个索引，可能的下一键锁包含以下间隔，圆括号表示不包括端点，方括号表示包括端点：

```mysql
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

对于最后的间隔，下一键锁锁住的间隙在索引中的最大值之上，上确界的值高于索引中的任何值，上确界不是一个真正的索引记录。所以实际上，下一键仅仅锁住最大值之后的间隙。

默认来说，InnoDB的事务隔离级别为 REPEATABLE READ。在这种情况下，InnoDB在查找和索引扫描时使用下一键锁，以防止幻读（参见 Section 14.5.4, “Phantom Rows”）

对于一个下一键锁的事务数据展示和通过SHOW ENGINE INNODB STATUS语句中是相似的，innoDB监视器输出如下：

```mysql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## 插入意向锁

一个插入意向锁是在 INSERT操作进行行插入之前的一种间隙锁定。这种锁表明尝试在这样的情况下进行插入，多个事务插入相同的索引间隔如果他们不是插入间隔中的相同位置，则不需要等待彼此。假设有4和7的索引记录，分别尝试插入5和6的事务在要获得要插入行的排它锁之前分别获得间隔为4和7之间的插入意向锁，但是不互相阻塞，因为要插入的行是不冲突的。

下面的例子显示一个事务在获得要插入记录的排它锁之前拿到插入意向锁。这个例子设置两个客户端A和B.

客户端A创建一张包含两个索引记录(90和102)的表，接着开启一个事务，在id>100的索引记录上放置一个排它锁，这个排它锁包含在记录102之前的一个间隔锁。

```mysql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```

客户端B开启一个事务在间隙中插入一条记录，该事务当他等待获取排它锁时，获得一个插入意向锁。

```mysql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```

对于一个插入意向锁的事务数据展示和通过SHOW ENGINE INNODB STATUS语句中是相似的，innoDB监视器输出如下：

```mysql
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```

## 自增锁(AUTO-INC Locks)

自增锁是一个特殊的表级锁，当事务插入带有AUTO_INCREMENT列的数据时获取，最简单的情况下，如果一个事务往表里插入数据，其他事务必须等待他们对该表的插入操作，让第一个事务的行插入接收到连续的主键值

innodb_autoinc_lock_mode 配置选项控制自增锁的算法。它允许你在自增值得可预测序列和插入操作最大并发之间权衡。

更多信息，参见 Section 14.8.1.5, “AUTO_INCREMENT Handling in InnoDB”.

(译者注，在插入行数不确定时才会出现，参考：http://keithlan.github.io/2017/03/03/auto_increment_lock/)

## 空间索引的谓词锁

InnoDB支持包含空间列的空间索引，参见Section 11.5.8, “Optimizing Spatial Analysis”

为了处理涉及空间索引的锁操作，下一键锁在事务隔离级别为REPEATABLE READ或者SERIALIZABLE工作的不是很好，在多维度的数据中，没有绝对的顺序概念，因此下一键锁中的下一键概念是不明确的。

为了能够支持有空间索引表的隔离级别。InnoDB使用谓词锁，一个空间索引包含最小边界矩形值(MBR), 因此InnoDB对于一个查询，通过在MBR值上放置谓词锁来保证强一致读，其他的事务不能插入或者修改与查询条件匹配结果中的一行。

###参考文档

https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html