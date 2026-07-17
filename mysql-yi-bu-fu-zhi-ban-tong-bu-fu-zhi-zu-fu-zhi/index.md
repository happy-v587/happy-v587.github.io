# mysql异步复制、半同步复制、组复制


> 本文由原作者归档自 CSDN，原文发布于 2021-03-03。原文链接：[mysql异步复制、半同步复制、组复制](https://blog.csdn.net/LIUHUAN0520/article/details/114334101)。

## 异步复制

sorce不管replica的死活，写进binlog后，commit完成就算成功。如果最后一个event没有发给replica，主库就挂了，那么就会有丢失数据的风险。 ![图 1](/images/mysql-yi-bu-fu-zhi-ban-tong-bu-fu-zhi-zu-fu-zhi.md/img-01.png)

## 半同步复制

通过官方的半同步插件，将binlog写完后，发送给replica，当replica写入到relay log后，在主库commit。这样可以最大情况保证数据能发送到replica。但是如果replica网络又问题，或者空间满了，导致ack返回时间慢、或者超时，这就会影响主库读写，所以还有个参数来控制`rpl_semi_sync_master_timeout`。 ![图 2](/images/mysql-yi-bu-fu-zhi-ban-tong-bu-fu-zhi-zu-fu-zhi.md/img-02.png)

## 组复制

官方在5.7.17以后开始正式提供。安装就像半同步插件一样简单。 ![图 3](/images/mysql-yi-bu-fu-zhi-ban-tong-bu-fu-zhi-zu-fu-zhi.md/img-03.png)

## 参考链接

- [https://dev.mysql.com/doc/refman/8.0/en/group-replication-primary-secondary-replication.html](https://dev.mysql.com/doc/refman/8.0/en/group-replication-primary-secondary-replication.html)
- [https://dev.mysql.com/doc/refman/8.0/en/group-replication-summary.html](https://dev.mysql.com/doc/refman/8.0/en/group-replication-summary.html)

