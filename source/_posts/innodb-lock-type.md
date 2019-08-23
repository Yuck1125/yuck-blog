---
title: InnoDB锁类型
date: 2019-08-22 17:28:43
tags: ["mysql","innodb"]
categories: "技术"
---
#### 1. Shared and Exclusive Locks
1. Shared Lock(简称S Lock，共享锁): 允许持有锁的事务读取行的操作
2. Exclusive Lock(简称 X Lock，排他锁): 允许持有锁的事务进行更新和删除行的操作

事务T1如果持有记录a的**S Lock**,此时事务t2也对记录a进行操作时，有两种情况：
* t2请求的是S Lock： t1,t2同时持有记录a的S Lock
* t2请求的是X Lock: t2会等待t1释放锁后，才能获取X Lock

事务t1如果持有是记录a的**X Lock**，那么t2不管请求S 还是X Lock,都要等t1释放锁后才能去请求。
#### 2. Intention Locks
**Intention Locks**是表级锁，它表示之后的事务需要获取哪种类型的行锁(S、X)。
*  **Intention Shared Lock(IS Lock):** 表示事务意图在表中各个行上加一个共享锁
*  **Intention Exclusive Lock (IX Lock):** 表示事务意图在表中各个行上加一个排他锁

我们可以通过`SELECT ... FOR SHARE ` 和 ` SELECT ... FOR UPDATE` 来获取IS、IX Lock。

意图锁有以下限制：
* 在事务可以获取表中某行的共享锁之前，它必须首先获取表上的**IS锁或更强的锁(IX、S、X)**。
* 在事务可以获取表中某行的独占锁之前，它必须首先获取表上的**IX锁**。

**表级锁**的兼容性如下表：

\ | X |	IX |	S	| IS
---|---|---|---|---
X  |	Conflict |	Conflict |	Conflict |	Conflict
IX |	Conflict |	**Compatible** |	Conflict |	**Compatible**
S  |	Conflict |	Conflict |	**Compatible** |	**Compatible**
IS |	Conflict |	**Compatible** |	**Compatible** |	**Compatible**

如果事务请求的锁和先有锁兼容，则获取到锁。否则，事务会等待，直到现有的锁被释放。
##### 2.1 SELECT ... FOR UPDATE
**T1**:
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test  for update;
+----+------+-------+
| id | name | name2 |
+----+------+-------+
|  1 | aa   | aa    |
|  2 | aA   | aA    |
+----+------+-------+
2 rows in set (0.03 sec)

 mysql>show engine innodb status;

------------
TRANSACTIONS
------------
Trx id counter 13049
Purge done for trx's n:o < 12996 undo n:o < 0 state: running but idle
History list length 113
LIST OF TRANSACTIONS FOR EACH SESSION:
...
---TRANSACTION 13048, ACTIVE 122 sec
2 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 19, OS thread handle 12056, query id 367 localhost ::1 root
```
**表里就两条记录，但有三把行锁 ？**
此时事务T2
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test for update;
Lock wait timeout exceeded; try restarting transaction


------------
TRANSACTIONS
------------
Trx id counter 13050
Purge done for trx's n:o < 12996 undo n:o < 0 state: running but idle
History list length 113
LIST OF TRANSACTIONS FOR EACH SESSION:
...
---TRANSACTION 13049, ACTIVE 11 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 20, OS thread handle 2708, query id 370 localhost ::1 root Sending data
select * from test for update
------- TRX HAS BEEN WAITING 11 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 153 page no 4 n bits 72 index PRIMARY of table `localtest`.`test` trx id 13049 lock_mode X waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bits 128
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 000000002e5a; asc     .Z;;
 2: len 7; hex 010000011a096e; asc       n;;
 3: len 2; hex 6161; asc aa;;
 4: len 2; hex 6161; asc aa;;

------------------
---TRANSACTION 13048, ACTIVE 458 sec
2 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 19, OS thread handle 12056, query id 367 localhost ::1 root
```

### 3. Record Locks
**Record Lock**锁的是索引。如果表没有，InnoDB会建一个隐藏的聚簇索引，并用该隐藏索引来锁定记录。
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test  where id=3 for update;
+----+------+-------+
| id | name | name2 |
+----+------+-------+
|  1 | aa   | aa    |
+----+------+-------+
1 row in set (0.03 sec)

------------
TRANSACTIONS
------------
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 6 row lock(s), undo log entries 1
MySQL thread id 13, OS thread handle 14940, query id 586 localhost ::1 root updating
update test set name='test' where id =3
------- TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 312 page no 4 n bits 96 index PRIMARY of table `localtest`.`test` trx id 18709
lock_mode X locks rec but not gap waiting
Record lock, heap no 19 PHYSICAL RECORD: n_fields 7; compact format; info bits 128

------------
TRANSACTIONS
------------
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 7 row lock(s), undo log entries 1
MySQL thread id 13, OS thread handle 14940, query id 589 localhost ::1 root updating
delete from test where id =3
Trx read view will not see trx with id >= 18711, sees < 18710
------- TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 312 page no 4 n bits 96 index PRIMARY of table `localtest`.`test` trx id 18709
lock_mode X locks rec but not gap waiting
Record lock, heap no 19 PHYSICAL RECORD: n_fields 7; compact format; info bits 128
```
### 4. Gap Locks

T1开启一个事务 并执行下面语句
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test;
+----+-----+
| id | num |
+----+-----+
|  1 |   1 |
|  3 |   3 |
|  5 |   5 |
|  9 |   9 |
| 10 |  10 |
| 15 |  15 |
| 20 |  20 |
+----+-----+
7 rows in set (0.03 sec)

mysql> select * from test where id between 5 and 10 for update;
+----+-----+
| id | num |
+----+-----+
|  5 |   5 |
|  9 |   9 |
| 10 |  10 |
+----+-----+
3 rows in set (0.03 sec)
```

T2
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test (id,num) values (100,100);
Query OK, 1 row affected (0.00 sec)

mysql> insert into test (id,num) values (4,4);
Query OK, 1 row affected (0.00 sec)

mysql> insert into test (id,num) values (6,6);
1205 - Lock wait timeout exceeded; try restarting transaction

····


MySQL thread id 20, OS thread handle 13996, query id 856 localhost ::1 root update
insert into test (id,num) values (6,6)
------- TRX HAS BEEN WAITING 42 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 313 page no 4 n bits 96 index PRIMARY
of table `localtest`.`test` trx id 20874 lock_mode X locks gap before rec
insert intention waiting
```

当`where xxx=?` 条件的`xxx`不是索引或者非唯一索引，会锁住前一个间隙
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test;
+----+-----+
| id | num |
+----+-----+
|  1 |   1 |
|  3 |   3 |
|  5 |   5 |
|  9 |   9 |
| 10 |  10 |
| 11 |  11 |
| 12 |  12 |
| 15 |  15 |
| 20 |  20 |
+----+-----+
9 rows in set (0.03 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test where num=15 for update;
+----+-----+
| id | num |
+----+-----+
| 15 |  15 |
+----+-----+
1 row in set (0.03 sec)
```
T2开启一个事务，并执行下面`insert`语句，

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test (id,num) values (17,17);
Query OK, 1 row affected (0.00 sec)

mysql> insert into test (id,num) values (16,16);
Query OK, 1 row affected (0.00 sec)

mysql> insert into test (id,num) values (14,14);
1205 - Lock wait timeout exceeded; try restarting transaction
mysql> insert into test (id,num) values (22,15);
1205 - Lock wait timeout exceeded; try restarting transaction
mysql> insert into test (id,num) values (23,12);
1205 - Lock wait timeout exceeded; try restarting transaction
····

MySQL thread id 20, OS thread handle 13996, query id 765 localhost ::1 root update
insert into test (id,num) values (14,14)
------- TRX HAS BEEN WAITING 41 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 313 page no 6 n bits 88 index num of table `localtest`.`test` trx id 20832
lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 12 PHYSICAL RECORD: n_fields 2;
compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
 1: len 4; hex 8000000f; asc     ;;

```
上面的情况锁住的区间是[12,15]

**检索条件必须有索引,没有索引的话，会锁定整张表所有的记录**
T1
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test;
+----+-----+
| id | num |
+----+-----+
|  1 |   1 |
|  3 |   3 |
|  5 |   5 |
|  9 |   9 |
| 10 |  10 |
| 15 |  15 |
| 20 |  20 |
+----+-----+
7 rows in set (0.03 sec)

mysql> select * from test where num=15 for update;
+----+-----+
| id | num |
+----+-----+
| 15 |  15 |
+----+-----+
1 row in set (0.04 sec)
```
T2
```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test (id,num) values (4,4);
1205 - Lock wait timeout exceeded; try restarting transaction
mysql> insert into test (id,num) values (100,100);
1205 - Lock wait timeout exceeded; try restarting transaction

```

```
MySQL thread id 20, OS thread handle 13996, query id 834 localhost ::1 root update
insert into test (id,num) values (4,4)
------- TRX HAS BEEN WAITING 24 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 313 page no 4 n bits 96 index PRIMARY of table `localtest`.`test` trx id 20872
lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 000000005114; asc     Q ;;
 2: len 7; hex 01000001242299; asc     $" ;;
 3: len 4; hex 80000005; asc     ;;
```


**PS:** 如果指定区间[5,10],没有值为5和10的记录。insert 数据的时候，gap locks 会扩大到就近的存在记录的范围。如扩大到[3,15]
### 5. Insert Intention Locks
不同数据插入到相同索引间隙不需要等待，互不影响。

### 6. Next-Key Locks
InnoDB在`REPEATABLE READ`事务隔离级别下，默认开启。其实就是record lock 和gap lock 组合使用。
官方文档写的范围
```
假设索引包含值10,11,13和20,

(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```
