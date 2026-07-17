# 不同数据库支持情况对比


> 本文由原作者归档自 CSDN，原文发布于 2021-04-01。原文链接：[不同数据库支持情况对比](https://blog.csdn.net/LIUHUAN0520/article/details/115369185)。

| SQL支持 | 事务支持 | 存储支持 | 集群支持 | 产品 |
| --- | --- | --- | --- | --- |
| Y | Y | 硬盘 | Y | [https://github.com/pingcap/tidb https://github.com/greenplum-db/gpdb https://github.com/cockroachdb/cockroach](https://github.com/pingcap/tidb) |
| Y | Y | 硬盘 | N | [https://github.com/mysql/mysql-server (inndob存储引擎)](https://github.com/mysql/mysql-server) |
| Y | Y | 内存 | Y | https://github.com/VoltDB/voltdb |
| Y | Y | 内存 | N | voltdb的单机版？ |
| Y | N | 硬盘 | Y | https://github.com/rqlite/rqlite |
| Y | N | 硬盘 | N | https://github.com/sqlite/sqlite |
| Y | N | 内存 | Y | ？ |
| Y | N | 内存 | N | [https://github.com/mysql/mysql-server (Memory存储引擎)](https://github.com/mysql/mysql-server) |
| N | Y | 硬盘 | Y | [https://github.com/distributedio/titan https://github.com/tikv/tikv](https://github.com/distributedio/titan) |
| N | Y | 硬盘 | N | tikv的单机版？ |
| N | Y | 内存 | Y | ？ |
| N | Y | 内存 | N | ？ |
| N | N | 硬盘 | Y | [https://github.com/influxdata/influxdb https://github.com/etcd-io/etcd https://github.com/mongodb/mongo](https://github.com/influxdata/influxdb) |
| N | N | 硬盘 | N | [https://github.com/ledisdb/ledisdb https://github.com/facebook/rocksdb](https://github.com/ledisdb/ledisdb) |
| N | N | 内存 | Y | [https://github.com/redis/redis https://github.com/CodisLabs/codis](https://github.com/redis/redis) |
| N | N | 内存 | N | https://github.com/redis/redis |

