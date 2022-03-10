---
title: MySql-Transaction-Isolation-Level
date: 2022-03-09 21:28:43
tags: MySql
---



# 事务的4大特性ACID
原子性（Atomicity）：一个事务中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。
一致性（Consistent）：一方面，在事务开始之前和事务结束以后，数据库的完整性没有被破坏；另一方面，写入的数据必须完全符合所有的预设规则。
隔离性（Isolation）：不同的会话或线程，操作数据库的时候可能产生多个事务。如果同时操作一张表或同一行数据，必然产生并发或干扰操作。隔离性要求事务间对表或数据操作是透明的，互相不存在干扰的，通过这种方式保证一致性。
持久性（Durable）：事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

#  MySQL 事务的隔离级别



## 准备环境

```
 docker pull mysql
 docker run --name=mysql -it -p3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
 docker ps -a
 docker exec -it 15084 bash
 mysql -u root -p
 mysql> create database kangpan
 mysql> use kangpan

```



## 准备数据

```
mysql> create table user(
    -> id int(10) auto_increment,
    -> name varchar(30) default null,
    -> age tinyint(4) default null,
    -> primary key (id)
    -> )engine=innodb charset=utf8mb4;

 insert into user(id, name, age) values (1,'kangpan',31);

mysql> select * from user;
+----+---------+------+
| id | name    | age  |
+----+---------+------+
|  1 | kangpan |   31 |
+----+---------+------+
1 row in set (0.00 sec)
```

## 事务并发可能出现的情况

#### 脏读（Dirty Read）

一个事务读到了另一个未提交事务修改过的数据



#### 幻读（Phantom）

一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来。（幻读在读未提交、读已提交、可重复读隔离级别都可能会出现）



#### 不可重复读（Non-Repeatable Read）

一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值。（不可重复读在读未提交和读已提交隔离级别都可能会出现）



#### 幻读（Phantom）

一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来。（幻读在读未提交、读已提交、可重复读隔离级别都可能会出现）



## 事务的隔离级别

MySQL的事务隔离级别一共有四个，分别是读未提交、读已提交、可重复读以及可串行化。

MySQL的隔离级别的作用就是让事务之间互相隔离，互不影响，这样可以保证事务的一致性。

隔离级别比较：可串行化>可重复读>读已提交>读未提交

隔离级别对性能的影响比较：可串行化>可重复读>读已提交>读未提交

由此看出，隔离级别越高，所需要消耗的MySQL性能越大（如事务并发严重性），为了平衡二者，一般建议设置的隔离级别为可重复读，MySQL默认的隔离级别也是可重复读。

#### 读未提交（READ UNCOMMITTED）

在读未提交隔离级别下，事务A可以读取到事务B修改过但未提交的数据。

可能发生脏读、不可重复读和幻读问题，一般很少使用此隔离级别。

#### 读已提交（READ COMMITTED）

在读已提交隔离级别下，事务B只能在事务A修改过并且已提交后才能读取到事务B修改的数据。

读已提交隔离级别解决了脏读的问题，但可能发生不可重复读和幻读问题，一般很少使用此隔离级别。

#### 可重复读（REPEATABLE READ）

在可重复读隔离级别下，事务B只能在事务A修改过数据并提交后，自己也提交事务后，才能读取到事务B修改的数据。可重复读隔离级别解决了脏读和不可重复读的问题，但可能发生幻读问题。

提问：为什么上了写锁（写操作），别的事务还可以读操作？

因为InnoDB有MVCC机制（多版本并发控制），可以使用快照读，而不会被阻塞。

#### 可串行化（SERIALIZABLE）

各种问题（脏读、不可重复读、幻读）都不会发生，通过加锁实现（读锁和写锁）。



## 隔离级别的实现原理

使用MySQL的默认隔离级别（可重复读）来进行说明。

每条记录在更新的时候都会同时记录一条回滚操作（回滚操作日志undo log）。同一条记录在系统中可以存在多个版本，这就是数据库的多版本并发控制（MVCC）。即通过回滚（rollback操作），可以回到前一个状态的值。

假设一个值从 1 被按顺序改成了 2、3、4，在回滚日志里面就会有类似下面的记录。

- read-view A : 回滚段 将 2 改成 1
- read-view B : 回滚段 将 3 改成 2

- read-view C : 当前值 4 

read-view A -> read-view B -> read-view C

当前值是 4，但是在查询这条记录的时候，不同时刻启动的事务会有不同的 read-view。在视图 A、B、C 里面，这一个记录的值分别是 1、2、4，同一条记录在系统中可以存在多个版本，就是数据库的多版本并发控制（MVCC）。对于 read-view A，要得到 1，就必须将当前值依次执行图中所有的回滚操作得到。

同时你会发现，即使现在有另外一个事务正在将 4 改成 5，这个事务跟 read-view A、B、C 对应的事务是不会冲突的。

提问：回滚操作日志（undo log）什么时候删除？

MySQL会判断当没有事务需要用到这些回滚日志的时候，回滚日志会被删除。

提问：什么时候不需要了？

当系统里么有比这个回滚日志更早的read-view的时候。

我们知道如果想要保证事务的原子性，就需要在异常发生时，对已经执行的操作进行**回滚**，在 MySQL 中，恢复机制是通过 **回滚日志（undo log）** 实现的，所有事务进行的修改都会先记录到这个回滚日志中，然后再执行相关的操作。如果执行过程中遇到异常的话，我们直接利用 **回滚日志** 中的信息将数据回滚到修改之前的样子即可！并且，回滚日志会先于数据持久化到磁盘上。这样就保证了即使遇到数据库突然宕机等情况，当用户再次启动数据库的时候，数据库还能够通过查询回滚日志来回滚将之前未完成的事务。

另外，MVCC 的实现依赖于：**隐藏字段、Read View、undo log**。在内部实现中，InnoDB 通过数据行的 DB_TRX_ID 和 Read View 来判断数据的可见性，如不可见，则通过数据行的 DB_ROLL_PTR 找到 undo log 中的历史版本。每个事务读到的数据版本可能是不一样的，在同一个事务中，用户只能看到该事务创建 Read View 之前已经提交的修改和该事务本身做的修改

## 查看当前会话隔离级别

#### 方式1

```
命令：SHOW VARIABLES LIKE 'transaction_isolation';

mysql> show variables like 'transaction_isolation';
+-----------------------+--------------+
| Variable_name  | Value |
+-----------------------+--------------+
| transaction_isolation | SERIALIZABLE |
+-----------------------+--------------+
```

#### 方式2

```
命令：SELECT @@transaction_isolation;

mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| SERIALIZABLE            |
+-------------------------+

mysql> select @@global.transaction_isolation;
+--------------------------------+
| @@global.transaction_isolation |
+--------------------------------+
| READ-UNCOMMITTED               |
+--------------------------------+
1 row in set (0.01 sec)
```

## 设置隔离级别

#### 方式1：通过set命令

```
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level;
其中level有4种值：
level: {
     REPEATABLE READ
   | READ COMMITTED
   | READ UNCOMMITTED
   | SERIALIZABLE
}
```

##### 关键词：GLOBAL

```
SET GLOBAL TRANSACTION ISOLATION LEVEL level;
eg: set global transaction isolation level read uncommitted;
* 只对执行完该语句之后产生的会话起作用
* 当前已经存在的会话无效
```

##### 关键词：SESSION

```
SET SESSION TRANSACTION ISOLATION LEVEL level;
* 对当前会话的所有后续的事务有效
* 该语句可以在已经开启的事务中间执行，但不会影响当前正在执行的事务
* 如果在事务之间执行，则对后续的事务有效。
```

##### 无关键词

```
SET TRANSACTION ISOLATION LEVEL level;
* 只对当前会话中下一个即将开启的事务有效
* 下一个事务执行完后，后续事务将恢复到之前的隔离级别
* 该语句不能在已经开启的事务中间执行，会报错的
```

#### 方式2：通过服务启动项命令

> 可以修改启动参数transaction-isolation的值
>
> 比方说我们在启动服务器时指定了--transaction-isolation=READ UNCOMMITTED，那么事务的默认隔离级别就从原来的REPEATABLE READ变成了READ UNCOMMITTED。



## 关于事务日志

关于事务日志的说明中，我们可以看得出来，只要修改的数据已经写入到日志并且持久化了，数据本身还没有写入磁盘时，即使断电了，系统在重启的时候依然会将数据恢复。那么我们再来看看官网给出的innodb_flush_log_at_trx_commit 参数的介绍
- 为0时，如果MySQL挂了或宕机可能会有1秒数据的丢失。
- 为1时， 只要事务提交成功，redo log记录就一定在硬盘里，不会有任何数据丢失。如果事务执行期间MySQL挂了或宕机，这部分日志丢了，但是事务并没有提交，所以日志丢了也不会有损失。
- 为2时， 只要事务提交成功，redo log buffer中的内容只写入文件系统缓存（page cache）。如果仅仅只是MySQL挂了不会有任何数据丢失，但是宕机可能会有1秒数据的丢失。

该属性主要是为数据库的ACID原则进行服务的，并且默认为1，但是实际情况下设置为2会提高很多的事务性能，设置1的时候，innodb的缓存会在事务提交或者每秒钟时都会进行磁盘的刷新操作，2的时候，innodb缓存会在提交事务时写入到事务日志但不会刷新磁盘，然后在每秒钟时进行磁盘刷新操作，2要比1提高很多性能，事务没有commit时，断电了，此时肯定数据是没有更新成功的，因为都还没有来得及写入事务日志，事务提交后，在写入事务日志的时候，发生断电，此时无论是参数的值是1还是2，都应该恢复不了数据了，每秒钟刷新磁盘时，发生断电，此时既然事务日志已经持久化了，那么重启后，数据是会自动恢复的。

#### 刷盘时机

InnoDB 存储引擎为 redo log 的刷盘策略提供了 innodb_flush_log_at_trx_commit 参数，它支持三种策略：

- **0** ：设置为 0 的时候，表示每次事务提交时不进行刷盘操作，后台线程进行刷盘
- **1** ：设置为 1 的时候，表示每次事务提交时都将进行刷盘操作（默认值）
- **2** ：设置为 2 的时候，表示每次事务提交时都只把 redo log buffer 内容写入 page cache

innodb_flush_log_at_trx_commit 参数默认为 1 ，也就是说当事务提交时会调用 fsync 对 redo log 进行刷盘

另外，InnoDB 存储引擎有一个后台线程，每隔1 秒，就会把 redo log buffer 中的内容写到文件系统缓存（page cache），然后调用 fsync 刷盘。也就是说，一个没有提交事务的 redo log 记录，也可能会刷盘。因为在事务执行过程 redo log 记录是会写入redo log buffer 中，这些 redo log 记录会被后台线程刷盘。除了后台线程每秒1次的轮询操作，还有一种情况，当 redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动刷盘。


#### 日志存储方式
硬盘上存储的 redo log 日志文件不只一个，而是以一个日志文件组的形式出现的，每个的redo日志文件大小都是一样的。
比如可以配置为一组4个文件，每个文件的大小是 1GB，整个 redo log 日志文件组可以记录4G的内容。
它采用的是环形数组形式，从头开始写，写到末尾又回到头循环写，在个日志文件组中还有两个重要的属性，分别是 write pos、checkpoint
- write pos 是当前记录的位置，一边写一边后移
- checkpoint 是当前要擦除的位置，也是往后推移
每次刷盘 redo log 记录到日志文件组中，write pos 位置就会后移更新。如果 write pos 追上 checkpoint ，表示日志文件组满了，这时候不能再写入新的 redo log 记录，MySQL 得停下来，清空一些记录，把 checkpoint 推进一下。

## 关于Autocommit

当变量autocommit的值为ON时，代表自动提交开启，改为OFF则变为手动提交。在手动提交模式下，可以使用下面两种指令开启事务：


```
start transaction;
begin;
```

结束事务的方式也有两种，事务确认提交
```
commit;
rollback;
```

```
mysql> set @@autocommit=0;

mysql> SHOW VARIABLES like '%autocommit%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set (0.00 sec)
```

## 悲观锁与乐观锁

- **悲观锁**：悲观锁指对数据被意外修改持保守态度，依赖数据库原生支持的锁机制来保证当前事务处理的安全性，防止其他并发事务对目标数据的破坏或破坏其他并发事务数据，将在事务开始执行前或执行中申请锁定，执行完后再释放锁定。这对于长事务来讲，可能会严重影响系统的并发处理能力

  ```
  LOCK TABLES a WRITE;
  INSERT INTO a VALUES (1,23),(2,34),(4,33);
  INSERT INTO a VALUES (8,26),(6,29);
  UNLOCK TABLES;
  ```

  锁定表可以加速用多个语句执行的INSERT操作，因为索引缓存区仅在所有INSERT语句完成后刷新到磁盘上一次。一般有多少INSERT语句即有多少索引缓存区刷新，如果能用一个语句插入所有的行，就不需要锁定；对于事务表，应使用BEGIN和COMMIT代替LOCK TABLES来加快插入

- **乐观锁**：乐观锁相对悲观锁而言，先假想数据不会被并发操作修改，没有数据冲突，只在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突了，则宣告失败，否则更新数据。这就要求避免使用长事务和锁机制，以免导致系统并发处理能力降低，保障系统生产效率。下面将说明使用乐观锁时的大致业务处理流程

  ```
  首 步：执行一次查询 select some_column as old_value from some_table where id = id_value (假设该值在当前业务处理过程中不会被其他并发事务修改)
  ...
  第n步：old_value参与中间业务处理，比如old_value被自己修改 new_value = f(old_value)。这期间可能耗时很长，但不会为持有 some_column 而申请所在的行或表锁定，因此其他并发事务可以获得该锁
  ...
  尾 步：执行条件更新 update some_table set some_column = new_value where id = id_value and some_column = old_value (条件更新中检查old_value是否被修改)
  ```

## 三大日志
MySQL InnoDB 引擎使用 redo log(重做日志) 保证事务的持久性，使用 undo log(回滚日志) 来保证事务的原子性。
MySQL数据库的数据备份、主备、主主、主从都离不开binlog，需要依靠binlog来同步数据，保证数据一致性。
