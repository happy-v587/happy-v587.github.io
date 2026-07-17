# mysqldump引发数据库不可读写的血案


> 本文由原作者归档自知乎，原文发布于 2021-01-26。原文链接：[mysqldump引发数据库不可读写的血案](https://zhuanlan.zhihu.com/p/347105199)。

## 问题

一直认为mysql在dump时候加上 --single-transaction 就不会影响业务，除非有DDL同时在操作同一张表。但是最近发现即是没有DDL也有锁表情况，慢日志记录详情如下：

```bash
# Time: 210115  3:05:10
# User@Host: sss[sss] @  [x.x.x.x]  Id: 6109323
# Query_time: 61.872232  Lock_time: 0.000000 Rows_sent: 0  Rows_examined: 0
SET timestamp=1610651110;
FLUSH /*!40101 LOCAL */ TABLES;
# Time: 210115  3:07:11
# User@Host: sss[sss] @  [x.x.x.x]  Id: 6109323
# Query_time: 120.989016  Lock_time: 0.000000 Rows_sent: 0  Rows_examined: 0
SET timestamp=1610651231;
FLUSH TABLES WITH READ LOCK;
```

当时使用的逻辑备份命令为

```bash
mysqldump -uroot -pxxx -R -E --single-transaction --skip-add-drop-table --set-gtid-purged=OFF --master-data=2 db_name
```

此时的疑问：是哪个过程导致mysqldump执行了，`FLUSH /*!40101 LOCAL */ TABLES;` 和 `FLUSH TABLES WITH READ LOCK;`命令？

## 原因

关于参数，查阅了官方的参数说明，找到了答案

```bash
  -E, --events        Dump events.
  -R, --routines      Dump stored routines (functions and procedures).
  --set-gtid-purged[=name]
                      Add 'SET @@GLOBAL.GTID_PURGED' to the output. Possible
                      values for this option are ON, OFF and AUTO. If ON is
                      used and GTIDs are not enabled on the server, an error is
                      generated. If OFF is used, this option does nothing. If
                      AUTO is used and GTIDs are enabled on the server, 'SET
                      @@GLOBAL.GTID_PURGED' is added to the output. If GTIDs
                      are disabled, AUTO does nothing. If no value is supplied
                      then the default (AUTO) value will be considered.
  --master-data[=#]   This causes the binary log position and filename to be
                      appended to the output. If equal to 1, will print it as a
                      CHANGE MASTER command; if equal to 2, that command will
                      be prefixed with a comment symbol. This option will turn
                      --lock-all-tables on, unless --single-transaction is
                      specified too (in which case a global read lock is only
                      taken a short time at the beginning of the dump; don't
                      forget to read about --single-transaction below). In all
                      cases, any action on logs will happen at the exact moment
                      of the dump. Option automatically turns --lock-tables
                      off.
  --single-transaction
                      Creates a consistent snapshot by dumping all tables in a
                      single transaction. Works ONLY for tables stored in
                      storage engines which support multiversioning (currently
                      only InnoDB does); the dump is NOT guaranteed to be
                      consistent for other storage engines. While a
                      --single-transaction dump is in process, to ensure a
                      valid dump file (correct table contents and binary log
                      position), no other connection should use the following
                      statements: ALTER TABLE, DROP TABLE, RENAME TABLE,
                      TRUNCATE TABLE, as consistent snapshot is not isolated
                      from them. Option automatically turns off --lock-tables.
```

此处看下 --master-data 参数说明：值为1会以非注释的方式展示位点信息；值为2时会通过注释的方式展示位点信息。

为了获取这样的位点信息，还是有一些副作用的。此流程会上：全局锁 `FLUSH TABLES WITH READ LOCK;`+关闭打开的表`FLUSH /*!40101 LOCAL */ TABLES;`，原来就是**--master-data**参数导致的。

那么有什么可以优化表锁时间吗？看情况，全局锁没有办法优化，表锁可以。

- 如果表是innodb存储引擎的可以通过添加`--single-transaction`参数来缓解表锁时间，但是在读写频繁的数据库上会把undo_log撑大也因此带来了弊端。
- 如果表是非innodb存储引擎，就只能被锁表了。数据量越大锁表时间越长，影响线上业务越久。

参数中说**--master-data**全局锁的时间会很短`in which case a global read lock is only taken a short time at the beginning of the dump`，但是根据开篇的慢日志记录可以看到，上锁的时间长达120秒。所以影响业务的情况应该包括：**上锁时间+锁表时间**。

## mysqldump实验

为了深入理解提到的参数作用，我们进行了下面5个实验。实验选择mysql这个系统库，并且开启general log来记录具体执行的sql

```bash
# 涉及到的参数
--master-data
--skip-lock-tables
--single-transaction
--lock-tables
```

## SQL0

> mysqldump -uroot -pxxx -R -E **--single-transaction** --skip-add-drop-table --set-gtid-purged=OFF **--master-data=2** demo

以上是本人一直使用的dump命令

![图 1](/images/mysqldump-yin-fa-shu-ju-ku-bu-ke-du-xie.md/img-01.jpg)

## SQL1

> mysqldump -uroot -pxxx -R -E --skip-add-drop-table --set-gtid-purged=OFF **--master-data=2** demo

![图 2](/images/mysqldump-yin-fa-shu-ju-ku-bu-ke-du-xie.md/img-02.jpg)

和上一组SQL对比，发现少了`START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */` `SAVEPOINT sp` `ROLLBACK TO SAVEPOINT sp`命令，这也表明了，没有通过事务的方式进行dump。并且没有显式的使用UNLOCK TABLES。这就说明在dump过程中一直在锁表，直到会话结束，全局锁自动释放。

## SQL2

> mysqldump -uroot -pxxx -R -E **--single-transaction** --skip-add-drop-table --set-gtid-purged=OFF demo

![图 3](/images/mysqldump-yin-fa-shu-ju-ku-bu-ke-du-xie.md/img-03.jpg)

去掉了**--master-data=2** 参数，我们没有看到 `FLUSH /*!40101 LOCAL */ TABLES`和`FLUSH TABLES WITH READ LOCK`。

## SQL3

> mysqldump -uroot -pxxxx -R -E --skip-add-drop-table --set-gtid-purged=OFF mysql

![图 4](/images/mysqldump-yin-fa-shu-ju-ku-bu-ke-du-xie.md/img-04.jpg)

去掉 **--single-transaction** 参数，我们看到mysqldump上了`lock tables xxx read`的表级别锁。

## SQL4

> mysqldump -uroot -pxxxx -R -E --skip-add-drop-table **--skip-lock-tables** --set-gtid-purged=OFF mysql

![图 5](/images/mysqldump-yin-fa-shu-ju-ku-bu-ke-du-xie.md/img-05.jpg)

加上**--skip-lock-tables**解决了 `LOCK TABLES a READ /*!32311 LOCAL */` `UNLOCK TABLES` 的问题

## 结论

通过上述实验，我们就可以清晰的知道每个参数的作用了。

## 参数

### --single-transaction

通过开启事务的方式进行dump

```bash
START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
SAVEPOINT sp
```

### --master-data=2

为了获取一致性位点，需要上两把锁

```bash
FLUSH /*!40101 LOCAL */ TABLES
FLUSH TABLES WITH READ LOCK
```

### --lock-tables

默认会锁住所有需要dump的表，来保证数据的一致性

```bash
LOCK TABLES `a` READ /*!32311 LOCAL */
LOCK TABLES `mysql.proc` READ /*!32311 LOCAL */
```

### --skip-lock-tables

和--lock-all-tables 的作用相反。

