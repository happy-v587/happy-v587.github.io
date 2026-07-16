# 恭喜，聪明的你发明了RocksDB的主从复制


> 本文由原作者归档自微信公众号「两分钟好奇心」，原文发布于 2025-01-06。原文链接：[恭喜，聪明的你发明了RocksDB的主从复制](https://mp.weixin.qq.com/s/YprloMHlrn0Nji92xxPaJg)。

## 背景

你是一个 RocksDB 进程，你的名字叫A。

![图 1](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-01.png)

（该图片来自rockdb.org）

很久很久以前，你承担着上层应用的 Put、Get 请求，你勤勤恳恳没有怨言。

突然有一天，你 Put、Get 的执行时间变长了，程序员们疯狂质问你，你为什么变慢了，你解释说是因为 Get 请求太多处理不过来了，要是能有其他 RocksDB 进程帮你承担一些请求就好了。

![图 2](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-02.png)

---

## 改进1

你灵机一动，想出了一个聪明的计划：将所有的写操作记录下来，就像人类记录历史一样。你创造了一个叫做binlog的文件，它像是你的日记，详细记录了你的每一个写操作。进程B，就像一个勤奋的学生，不断地阅读这本日记，就像通过`tail -f`命令实时地学习你的新知识，并将这些知识应用到自己的数据库中。看着进程B如此努力，你忍不住露出了满意的微笑。

![图 3](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-03.png)

---

## 改进2

随着时间的流逝，你的日记binlog变得越来越厚重，就像是一本越来越厚的百科全书。你注意到，旧的章节（早期的写操作）已经过时，不再被需要了。于是你决定，每当日记达到512MB时，就翻开新的一页，开始一个新的文件。同时，你还设立了一个周期性的清理计划，定期删除那些古老的故事，保持你的书架整洁有序。

![图 4](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-04.png)

---

## 改进3

不久后，你注意到即使把过去的故事清理掉了，存储空间还是越来越紧张。这不是因为binlog，而是因为你和进程B都在保存着一模一样的数据库数据，这种冗余让你感到有些浪费。你开始思考如何能够共享这些数据，而不是每个人都保存一份。你决定，是时候推翻旧有的架构，采用一种更为高效的数据共享方式了。

![图 5](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-05.png)

(事实上MySQL主从复制就是这种架构)

---

## 改进4

你采取了一项大胆的措施，直接将自己的数据库目录共享给进程B，让进程B能够直接访问你的数据库文件，但进程B会限制自己只能以只读方式操作。这样一来，B进程可以直接读取你的数据，而不需要再维护一份自己的副本，从而节省了大量的存储空间。

![图 6](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-06.png)

---

## 改进5

然而，你很快发现了一个问题：每当你更新了数据，进程B并不能立即看到这些变化。为了解决这个问题，你尝试了一种简单粗暴的方法——重启进程B，强迫它重新加载数据，以便能看到最新的状态。但这种方法意味着每次同步都会有一段时间服务不可用，这显然是不可接受的。

![图 7](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-07.png)

---

## 改进6

为了彻底解决这个问题，你发明了一个更加精巧的同步机制。你利用RocksDB的WAL（预写日志）和MANIFEST文件追加写的性质。你开发了一个工具，周期性地读取这些文件，只同步那些自上次同步以来发生变化的数据。这样，进程B就可以在不中断服务的情况下，渐进式地更新数据，实现了零停机时间的数据同步。你自豪地看着你的这个新系统运行得如此顺畅，知道这将是一个新的开始。

![图 8](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-08.png)

---

## 当前 RocksDB 主从复制架构

RocksDB数据分为内存和磁盘两部分。内存层面：由多个Memtable构成；磁盘层面：由WAL/MANINFEST/LOG/SST文件构成。

其中WAL和Memtable是同分异构的，本质是一份数据。WAL会随着kv写操作的调用不断追加写入；Memtable是在内存中对key做了排序处理便于快速找到目标key。

Memtable 和 SST也是同分异构的，SST 是 key 有序的 Memtable 持久化到磁盘的文件格式。SST 一般数 MB、数据顺序写入、写完后不可变更。

目前RocksDB的从复制实现逻辑为：

- **启动：** 按照只读模式，访问数据目录，根据MANINFEST恢复元信息、根据WAL恢复Memtable（还未持久化到磁盘那些）

- **复制：** 提供了catchup函数，通过周期性调用实现数据的同步。同步的内容为：MANINFEST 增量部分继续完善元数据（完善期间判断发现有Memtable已经持久化到磁盘，那么对应的Memtable会被delete）、WAL增量部分继续重建 Memtable。

- **Get请求：** RocksDB请求流程会先在 Memtable 找数据，找不到就在 SST 文件中找。因为 catchup WAL保证了 Memtable 的完整性，catchup MANINFEST保证了元信息的完整性，并且SST具有不可更改性，所以从上的Get请求跟主处理流程一样。

![图 9](/images/gong-xi-cong-ming-de-ni-fa-ming-le-rocksdb-de-zhu-cong-fu-zhi.md/img-09.png)

https://github.com/facebook/rocksdb/wiki/Read-only-and-Secondary-instances

