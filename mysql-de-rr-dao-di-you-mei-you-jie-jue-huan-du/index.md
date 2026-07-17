# MySQL 的 RR 到底有没有解决幻读


> 本文由原作者归档自知乎，原文发布于 2026-06-15。原文链接：[MySQL 的 RR 到底有没有解决幻读](https://zhuanlan.zhihu.com/p/2049918197281952379)。

## MySQL 的 RR 到底有没有解决幻读？

关于数据库隔离级别，有一个流传很多年的结论：Read Committed 解决脏读，Repeatable Read 解决不可重复读，Serializable 解决幻读。

所以很多人的认知是：RR 解决不了幻读。

但如果你查 MySQL 官方文档或者一些数据库专家的分享，又会看到另外一种说法：InnoDB 的 RR 已经解决了幻读。

那么问题来了：**到底是谁说错了？**

事实上，两边都没错。因为讨论这个问题之前，必须先弄清楚一个经常被忽略的前提：你执行的到底是快照读，还是当前读？

---

## 幻读到底是什么？

先看一个经典场景。

假设当前 `orders` 表中有三条数据：

```text
id | amount
---|-------
1  | 80
2  | 150
3  | 200
```

| 时间线 | 事务A | 事务B |
| --- | --- | --- |
| t1 | SELECT * FROM orders WHERE amount > 100; |  |
|  | （返回两条记录） |  |
| t2 |  | BEGIN; |
| t3 |  | INSERT INTO orders VALUES(4, 180); |
| t4 |  | COMMIT; |
| t5 | SELECT * FROM orders WHERE amount > 100; |  |
|  | （返回三条记录） |  |

同样的查询条件，两次查询结果集却发生了变化，这就是所谓的幻读（Phantom Read）。

ANSI SQL 对幻读的定义其实很简单：

> 同一事务中，两次相同条件的查询返回了不同的结果集。

**注意这里说的是“结果集变化”，并没有规定必须是什么 SQL，也没有规定底层实现方式。**

---

## 为什么很多人觉得 RR 已经解决了幻读？

因为他们做的实验通常是这样的：

| 时间线 | 事务A | 事务B |
| --- | --- | --- |
| t1 | BEGIN; |  |
| t2 | SELECT * FROM t; |  |
| t3 |  | BEGIN; |
| t4 |  | INSERT INTO t VALUES(4); |
| t5 |  | COMMIT; |
| t6 | SELECT * FROM t; |  |

`t6` 结果仍然只有原来的数据，看不到 `id=4`。

于是很多人得出结论：

> RR 下不会出现幻读。

这个实验本身没有问题，但它验证的其实是 **MVCC 的快照读能力**。

在 InnoDB 的 RR 隔离级别下，第一次普通 `SELECT` 会创建一个 Read View，后续查询都基于同一个快照进行读取，因此事务A始终看到的是事务开始时的数据状态。

从这个角度来说：对于快照读，幻读确实不会出现。

---

## 那为什么标准教材又说 RR 解决不了幻读？

因为标准讨论的是事务现象，而不是 MySQL 的具体实现。更重要的是，InnoDB 里除了快照读，还有另一种读取方式：

```text
SELECT ... FOR UPDATE;
UPDATE ...
DELETE ...
```

这些操作属于当前读（Current Read）。当前读不会读取历史快照，而是直接读取最新版本的数据。

于是我们把实验稍微改一下。

| 时间线 | 事务A | 事务B |
| --- | --- | --- |
| t1 | BEGIN; |  |
| t2 | SELECT * FROM t; |  |
| t3 |  | BEGIN; |
| t4 |  | INSERT INTO t VALUES(4); |
| t5 |  | COMMIT; |
| t6 | SELECT * FROM t FOR UPDATE; |  |

结果会发现：

```text
1
2
3
4
```

新插入的记录出现了。从事务A的角度看：

- 第一次查询看到 3 条记录
- 第二次查询看到 4 条记录

按照 ANSI SQL 的定义，这就是幻读。

---

## 真正决定幻读是否出现的，其实不是 RR，而是第一次访问时有没有加锁

再看另外一个实验。

| 时间线 | 事务A | 事务B |
| --- | --- | --- |
| t1 | BEGIN; |  |
| t2 | SELECT * FROM t FOR UPDATE; |  |
|  | （此时 InnoDB 会加 Next-Key Lock） |  |
| t3 |  | BEGIN; |
| t4 |  | INSERT INTO t VALUES(4); |
|  |  | （会被阻塞，直到事务A提交） |
| t5 | SELECT * FROM t FOR UPDATE; |  |
|  | （这时候事务A无论再执行多少次，结果都不会发生变化，因为新的记录根本插不进来） |  |

---

## InnoDB 到底是怎么解决幻读的？

很多文章把原因简单归结为 MVCC。实际上并不准确。InnoDB 能够避免幻读，本质上依赖的是两套机制：

| 机制 | 说明 | 效果 |
| --- | --- | --- |
| MVCC | 负责解决快照读问题。保证事务内多次普通 SELECT 看到的是同一个一致性视图。 | MVCC 让你看不到幻影 |
| Next-Key Lock | 负责解决当前读问题。防止其他事务向查询范围内插入新的记录。 | Next-Key Lock 让别人造不出幻影。 |

两者配合，才构成了 InnoDB 在 RR 隔离级别下的完整并发控制方案。

---

## 所以 RR 到底解决了幻读吗？

我觉得更准确的答案应该是：RR 本身并不能简单地说“解决”或者“没解决”幻读，而要看具体使用的是快照读还是当前读。

对于普通 `SELECT`：

- InnoDB 通过 MVCC 保证一致性视图；
- 同一个事务内看到的是同一个快照；
- 不会观察到幻读。

对于 `FOR UPDATE`、`UPDATE`、`DELETE` 等当前读：

- InnoDB 依赖 Gap Lock 和 Next-Key Lock；
- 防止其他事务向查询范围插入新记录；
- 从而避免幻读。

但如果先进行了快照读，后面又执行当前读，那么仍然有机会观察到“幻影记录”。

这也是为什么有人说：RR 解决了幻读。

而另一些人又坚持：RR 没有解决幻读。

因为他们讨论的根本不是同一种场景。

---

## 一个更有意思的问题

看到这里，其实还有一个问题值得继续思考。

MySQL 为了避免幻读，引入了：

- MVCC
- Gap Lock
- Next-Key Lock

那么 PostgreSQL 呢？PostgreSQL 也有 MVCC，但没有 InnoDB 这种 Gap Lock。它是怎么处理幻读的？Oracle、SQL Server、TiDB 的行为又是否一样？

下一篇，我们来聊聊主流数据库在幻读问题上的不同设计思路，以及为什么有些数据库甚至不需要 Gap Lock，也能实现 Serializable。

