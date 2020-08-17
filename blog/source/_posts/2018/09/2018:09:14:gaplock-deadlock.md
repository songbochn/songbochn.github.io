---
title: mysql-gap锁引发死锁原因探寻
date: 2018-09-14 14:56:18
tag:
	- mysql
	- deadlock
	- lock
catergory:
	- 数据库
---

起因是同事描述发现一起mysql死锁现象，于是饶有兴趣的排查起来,将sql中无关值或字段用xxx代替

#### 1.排查过程：
发生死锁后，首先登录机器通过

```
show engine innodb status;
```
查询死锁信息.
<!--more-->

```mysql
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-09-13 12:52:57 0x7f84f516a700
*** (1) TRANSACTION:
TRANSACTION 8419693033, ACTIVE 0 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 1
MySQL thread id 11256043, OS thread handle 140199339202304, query id 3102245836 172.18.210.20 accounting update
/*id:8b9b0ea6*/insert into subject_ledger (subject_code, xxx,
        xxx, xxx, xxx,
        xxx, xxx, xxx,
        xxx, xxx, xxx,
        accounting_date, xxx, xxx,
        xxx)
        values (1122010120, xxx,
        xxx, xxx,
        xxx,
        xxx, xxx, xxx,
        xxx, xxx, xxx,
        '2018-09-13 00:00:00', xxx, xxx, xxx
        )
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 229883 page no 4 n bits 680 index uk_date_subject of table `accounting`.`subject_ledger` trx id 8419693033 lock mode S waiting
*** (2) TRANSACTION:
TRANSACTION 8419693029, ACTIVE 0 sec inserting
mysql tables in use 1, locked 1
9 lock struct(s), heap size 1136, 41 row lock(s), undo log entries 14
MySQL thread id 11256040, OS thread handle 140209024313088, query id 3102246333 172.18.210.20 accounting update
/*id:8b9b0ea6*/insert into subject_ledger (subject_code, xxx,
        xxx, xxx, xxx,
        xxx, xxx, xxx,
        xxx, xxx, xxx,
        accounting_date, xxx, xxx,
        xxx)
        values (22410104, xxx,
        xxx, x,
        xxx,
        xxx, xxx, xxx,
        xxx, xxx, xxx,
        '2018-09-13 00:00:00', xxx, xxx, xxx
        )
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 229883 page no 4 n bits 680 index uk_date_subject of table `accounting`.`subject_ledger` trx id 8419693029 lock_mode X locks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 229883 page no 4 n bits 680 index uk_date_subject of table `accounting`.`subject_ledger` trx id 8419693029 lock_mode X locks gap before rec insert intention waiting
*** WE ROLL BACK TRANSACTION (1)
```

其中uk_date_subject为subject_code和accounting_date组成的唯一键索引。

在日志中，对应关系如下:

|日志| 锁| 备注|
|:-- |---|---|
|lock_mode X locks rec but not gap|记录锁（LOCK_REC_NOT_GAP）||
|lock_mode X locks gap before rec|间隙锁（LOCK_GAP）||
|lock_mode X|Next-key 锁（LOCK_ORNIDARY）||
|lock_mode X locks gap before rec insert intention|插入意向锁（LOCK_INSERT_INTENTION）||

这里有一点要注意的是，并不是在日志里看到 lock_mode X 就认为这是 Next-key 锁，因为还有一个例外：如果在 supremum record 上加锁，locks gap before rec 会省略掉，间隙锁会显示成 lock_mode X，插入意向锁会显示成 lock_mode X insert intention。

锁可以分为几种维度，互不冲突， 锁的类型（X,S）和锁的范围(记录锁, 间隙锁 ,临键锁，表记锁

意向锁的含义是如果对一个结点加意向锁，则说明该结点的下层结点正在被加锁；对任一结点加锁时，必须先对他的上层结点加意向锁；如：对表中的任一行加锁时，必须先对他所在的表加意向锁，然后在对该行加锁，这样事务对表加锁时，就不再需要检查表中每行记录的锁标志位了，系统系率大大提高

插入意向锁是带有插入意向的间隙锁

那么整理如下：

| 事务Id     | sql                                                          | 持有锁                                      | 等待锁                                              | 备注 |
| ---------- | ------------------------------------------------------------------------ | ------------------------------------------- | --------------------------------------------------- | ---- |
| 8419693033 | insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>accounting_date, xxx, xxx,<br>xxx)<br>values (1122010120, xxx,<br>xxx, xxx,<br>xxx,<br>xxx, xxx,xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00', xxx, xxx, xxx<br>) |                                             | 共享临键锁<br>（LOCK X \|LOCK_ORNIDARY)             |      |
| 8419693029 | insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx,xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>accounting_date, xxx, xxx,<br>xxx)<br>values (22410104, xxx,<br>xxx, xxx,<br>xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00',xxx, xxx, xxx<br>) | 记录排他锁<br>（LOCK X \|LOCK_REC_NOT_GAP） | 插入意向排他锁<br>（LOCK X \|LOCK_INSERT_INTENTION) |      |

分析记录如下：

| 事务一                                                       | 事务二                                                       | 分析                                                         | 备注 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ---- |
| begin                                                        | begin                                                        |                                                              |      |
                                                                                                                                                                                            | insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx, xxx,<br>xxx, xxx,xxx,<br>xxx,xxx, xxx,<br>accounting_date, xxx,xxx,<br>xxx)<br>values(1122010120, xxx,<br>xxx, xxx,<br>xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00', xxx, xxx, xxx<br>) | 获得唯一键为(1122010120，2018-09-13 00:00:00)记录索引的X锁<br/>获得锁过程 IX(表级锁) ->插入意向锁（gap Lock）-> X                                                             |      |
| insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>xxx,xxx,xxx,<br>accounting_date,xxx,xxx,<br>xxx)<br>values(1122010120,xxx,<br>xxx, xxx,<br>xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00', xxx, xxx, xxx<br>) |                                                              | 由于事务二插入成功后，获得X锁，当事务一插入同一条记录的时候检测到潜在唯一键冲突，会申请获得S next Key 锁，但是由于唯一键为(1122010120，2018-09-13 00:00:00)的X锁已经被事务二拿到，因此等待获取，进入锁等待队列<br>锁记录过程 IS -> S ->IX ->插入意向锁-> X |      |
|                                                              | insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>accounting_date, xxx, xxx,<br>xxx)<br>values (22410104, xxx,<br>xxx, xxx,<br>xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00', xxx, xxx, xxx<br>) | 此时再插入唯一键为(22410104，2018-09-13 00:00:00)的记录，首先要获得IX锁，然后获得插入意向锁，在进行插入。插入意向锁是一个锁类型为gap lock的锁，由于它要插入的记录位于事务一申请锁的gap内，获取不到插入意向锁，等待，此时造成死锁。<br> |      |
|                                                              |                                                              | 形成锁循环，事务一等事务二的X锁，事务二等事务一的S锁         |      |



####特殊疑点如下：

#####1.为什么事务一要去拿S next key锁？

由于事务二已经取得这条记录的X锁，事务一在重复插入这条记录的时候，Mysql会检测到潜在唯一键冲突，因此会加上S锁，保证这条记录不被别人修改，然后进行判定。这是官方文档介绍:

```mysql
INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock,
not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.

Prior to inserting the row, a type of gap lock called an insert intention gap lock is set.
This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other
if they are not inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions
that attempt to insert values of 5 and 6 each lock the gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock
on the inserted row, but do not block each other because the rows are nonconflicting.

If a duplicate-key error occurs, a shared lock on the duplicate index record is set.
This use of a shared lock can result in deadlock should there be multiple sessions trying to insert the same row if another session
already has an exclusive lock. This can occur if another session deletes the row. Suppose that an InnoDB table t1 has the following structure:
```

为什么是next key锁，

因为事务一试图在一条有可能不存在的记录加S锁（事务二可能回滚），此时Mysql做法是加一个next key锁锁定一个范围。这是因为在rr级别下当要对一条记录加锁，但记录不存在，锁定范围避免幻读。

#####2.为什么插入意向锁会和gap锁冲突

首先gap锁和gap锁之间是不会冲突的，无论这个锁的类型是S还是X，而插入意向锁实际上就是一个gap锁

```mysql
An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion.
```

按理说不会冲突才对，但是官方文档如下

```mysql
Gap locks in InnoDB are “purely inhibitive”, which means that their only purpose is to prevent other transactions from inserting to the gap.
Gap locks can co-exist. A gap lock taken by one transaction does not prevent another transaction from taking a gap lock on the same gap.
There is no difference between shared and exclusive gap locks. They do not conflict with each other, and they perform the same function.
```

也就是说gap锁的唯一目的就是阻止别人插入这个gap，但是插入意向锁是带有插入意向的间隙锁，也就是当我给一个范围加上gap锁后，那么你要获取插入意向锁(间隙锁)时，会被gap锁阻止，因为gap锁的目的就是不允许别人在该范围插入。

#####3.插入唯一键为(22410104，2018-09-13 00:00:00)的记录落在事务一所获取S的区间内了吗？

换句话来说如果我把这个唯一键改成不在事务一锁的范围内就不会产生死锁了。

我们所常见范围举例都一个主键id值，类似(1,5)这个范围等等。

这里的唯一索引是subject_code 和accountint_date组成的，由于先锁二级索引，再去锁聚簇索引(主键索引)，所以对这个范围判断很模糊，需要根据实验来看：

下面开始实验，开始建立与测试环境相同的表结构,此时是空表

然后做以下操作

| 分析序号(用作记录分析) | 事务一                                                       | 事务二                                                       |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1                      | begin;                                                       | begin;                                                       |
| 2                      |                                                              | insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>accounting_date, xxx, xxx,<br>xxx)<br>values (1122010120, xxx,<br>xxx, xxx,<brxxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00', xxx, xxx, xxx<br>) |
| 3                      |                                                              |                                                              |
| 4                      | insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>accounting_date, xxx, xxx,<br>xxx)<br>values (1122010120, xxx,<br>xxx, xxx,<br>xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00', xxx, xxx, xxx<br>) |                                                              |
| 5                      |                                                              |                                                              |
| 6                      |                                                              | insert into subject_ledger (subject_code, xxx,<br/>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>accounting_date, xxx, xxx,<br>xxx)<br>values (22410104, xxx,<br>xxx, x,<br>xxx,<br>xxx, xxx, xxx,<br>xxx, xxx, xxx,<br>'2018-09-13 00:00:00', xxx, xxx, xxx<br>) |
| 7                      |                                                              |                                                              |



第5步时

看一下show engine innodb status信息:

```mysql
------------

TRANSACTIONS

------------

Trx id counter 1925

Purge done for trx's n:o < 1919 undo n:o < 0 state: running but idle

History list length 118

LIST OF TRANSACTIONS FOR EACH SESSION:

---TRANSACTION 281479574209872, not started

0 lock struct(s), heap size 1136, 0 row lock(s)

---TRANSACTION 281479574208968, not started

0 lock struct(s), heap size 1136, 0 row lock(s)

---TRANSACTION 281479574208064, not started

0 lock struct(s), heap size 1136, 0 row lock(s)

---TRANSACTION 1924, ACTIVE 9 sec inserting

mysql tables in use 1, locked 1

LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1

MySQL thread id 9, OS thread handle 123145436614656, query id 132 localhost root update

insert into subject_ledger (subject_code, xxx,

        xxx, xxx, xxx,

       xxx, xxx, xxx,

        xxx, xxx, xxx,

        accounting_date, xxx, xxx,

        xxx)

        values (1122010120, xxx,

        xxx, xxx,

        xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        '2018-09-13 00:00:00', xxx, xxx, xxx

        )

------- TRX HAS BEEN WAITING 9 SEC FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 31 page no 4 n bits 72 index uk_date_subject of table `mydata`.`subject_ledger` trx id 1924 lock mode S waiting

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0

 0: len 5; hex 99a0da0000; asc      ;;

 1: len 8; hex 0000000042e08408; asc     B   ;;

 2: len 8; hex 00000000000000a8; asc         ;;

------------------
```



------------------

select * from information_schema.innodb_LOCKS; 信息：

```mysql
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
| lock_id     | lock_trx_id | lock_mode | lock_type | lock_table                | lock_index      | lock_space | lock_page | lock_rec | lock_data                |
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
| 1924:31:4:2 | 1924        | S         | RECORD    | `mydata`.`subject_ledger` | uk_date_subject |         31 |         4 |        2 | 0x99A0DA0000, 1122010120 |
| 1923:31:4:2 | 1923        | X         | RECORD    | `mydata`.`subject_ledger` | uk_date_subject |         31 |         4 |        2 | 0x99A0DA0000, 1122010120 |
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)
```


select * from information_schema.innodb_lock_waits信息:

```mysq
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 1924              | 1924:31:4:2       | 1923            | 1923:31:4:2      |
+-------------------+-------------------+-----------------+------------------+
1 row in set, 1 warning (0.01 sec)

```

第7步产生死锁:

再看show engine innodb status信息:

```mysql
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-09-14 10:58:20 0x700008059000
*** (1) TRANSACTION:
TRANSACTION 1924, ACTIVE 11 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 9, OS thread handle 123145436614656, query id 132 localhost root update
insert into subject_ledger (subject_code, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        accounting_date, xxx, xxx,
                         xxx)

        values (1122010120, xxx,

        xxx, xxx,

        xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        '2018-09-13 00:00:00', xxx, xxx, xxx

        )
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 31 page no 4 n bits 72 index uk_date_subject of table `mydata`.`subject_ledger` trx id 1924 lock mode S waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 5; hex 99a0da0000; asc      ;;
 1: len 8; hex 0000000042e08408; asc     B   ;;
 2: len 8; hex 00000000000000a8; asc         ;;

*** (2) TRANSACTION:
TRANSACTION 1923, ACTIVE 39 sec inserting
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 2
MySQL thread id 10, OS thread handle 123145436893184, query id 136 localhost root update
insert into subject_ledger (subject_code, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        accounting_date, xxx, xxx,

        xxx)

        values (22410104, xxx,

        xxx, xxx,

        xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,
        '2018-09-13 00:00:00', xxx, xxx, xxx
        )
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 31 page no 4 n bits 72 index uk_date_subject of table `mydata`.`subject_ledger` trx id 1923 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 5; hex 99a0da0000; asc      ;;
 1: len 8; hex 0000000042e08408; asc     B   ;;
 2: len 8; hex 00000000000000a8; asc         ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 31 page no 4 n bits 72 index uk_date_subject of table `mydata`.`subject_ledger` trx id 1923 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 5; hex 99a0da0000; asc      ;;
 1: len 8; hex 0000000042e08408; asc     B   ;;
 2: len 8; hex 00000000000000a8; asc         ;;

*** WE ROLL BACK TRANSACTION (1)
```


其余两个select * from information_schema.innodb_LOCKS和select * from information_schema.innodb_lock_waits查询结果空。

第5步可以看到锁的范围为(0x99A0DA0000, 1122010120)

关于这个16进制值，google不到，猜测是一个地址表示下边界即负无穷

我们第二次所锁的subject_code为22410104,看起来符合我们的预期，确实落在区间里了，那我们把第二次subject_code改一下再看，改成1122010121看看会不会死锁

于是清表再来，

过程如上不写，只是改了第6步sql的值

来看第五步

```mysql
--TRANSACTION 1949, ACTIVE 6 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 9, OS thread handle 123145436614656, query id 203 localhost root update
insert into subject_ledger (subject_code, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        accounting_date, xxx, xxx,

        xxx)

        values (1122010120, xxx,

        xxx, xxx,

        xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        '2018-09-13 00:00:00', xxx, xxx, xxx

        )
------- TRX HAS BEEN WAITING 6 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 31 page no 4 n bits 72 index uk_date_subject of table `mydata`.`subject_ledger` trx id 1949 lock mode S waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
0: len 5; hex 99a0da0000; asc      ;;
 1: len 8; hex 0000000042e08408; asc     B   ;;
 2: len 8; hex 00000000000000ab; asc         ;;

------------------
```

select * from information_schema.innodb_LOCKS; 信息：

```mysql
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
| lock_id     | lock_trx_id | lock_mode | lock_type | lock_table                | lock_index      | lock_space | lock_page | lock_rec | lock_data                |
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
| 1949:31:4:2 | 1949        | S         | RECORD    | `mydata`.`subject_ledger` | uk_date_subject |         31 |         4 |        2 | 0x99A0DA0000, 1122010120 |
| 1948:31:4:2 | 1948        | X         | RECORD    | `mydata`.`subject_ledger` | uk_date_subject |         31 |         4 |        2 | 0x99A0DA0000, 1122010120 |
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)
```

select * from information_schema.innodb_lock_waits信息:

```mysql
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 1949              | 1949:31:4:2       | 1948            | 1948:31:4:2      |
+-------------------+-------------------+-----------------+------------------+
1 row in set, 1 warning (0.00 sec)
```

第6步sql插入成功，没有死锁发生：

第7步事务一等锁超时，事务自动撤销

此时:

show engine innodb status:

```mysql
---TRANSACTION 1949, ACTIVE 6 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 9, OS thread handle 123145436614656, query id 203 localhost root update
insert into subject_ledger (subject_code, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        accounting_date, xxx, xxx,

        xxx)

        values (1122010120, xxx,

        xxx, xxx,

        xxx,

        xxx, xxx, xxx,

        xxx, xxx, xxx,

        '2018-09-13 00:00:00', xxx, xxx, xxx

        )
------- TRX HAS BEEN WAITING 30 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 31 page no 4 n bits 72 index uk_date_subject of table `mydata`.`subject_ledger` trx id 1949 lock mode S waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
0: len 5; hex 99a0da0000; asc      ;;
 1: len 8; hex 0000000042e08408; asc     B   ;;
 2: len 8; hex 00000000000000ab; asc         ;;

------------------
```

select * from information_schema.innodb_LOCKS; 信息：

```mysql
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
| lock_id     | lock_trx_id | lock_mode | lock_type | lock_table                | lock_index      | lock_space | lock_page | lock_rec | lock_data                |
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
| 1949:31:4:2 | 1949        | S         | RECORD    | `mydata`.`subject_ledger` | uk_date_subject |         31 |         4 |        2 | 0x99A0DA0000, 1122010120 |
| 1948:31:4:2 | 1948        | X         | RECORD    | `mydata`.`subject_ledger` | uk_date_subject |         31 |         4 |        2 | 0x99A0DA0000, 1122010120 |
+-------------+-------------+-----------+-----------+---------------------------+-----------------+------------+-----------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)
```

select * from information_schema.innodb_lock_waits信息:

```mysql
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 1949              | 1949:31:4:2       | 1948            | 1948:31:4:2      |
+-------------------+-------------------+-----------------+------------------+
1 row in set, 1 warning (0.00 sec)
```

与第五步相比没有变化

可以发现确实是在事务一锁的范围内；

####4.探究多列唯一索引的范围划分

在上文中多列索引有accounting_date和subject_code组成，由于那三条sql的accounting_date是一样的，所以subject_code划分没有问题，那如果accounting也不一样呢？

清表再来

此时让第6步sql的subject_code维持原来的22410104，把记账日期改为(2018-09-12 00:00:00)观察范围，

产生死锁,锁的范围是(0x99A0DA0000, 1122010120)，原因与发生死锁的原因相同

日期改成2018-09-14 00:00:00呢?

清表再来;

未发生死锁，与把subject_code改为1122010121的结果相同

那么其实可以总结如下

**当多列都在范围内，（是否包括边界取决于，其他事务对这个范围加的是gap lock 还是next key lock，）才会出现冲突，只要有一列不符合条件都不会发生死锁。**

