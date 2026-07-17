# 深度结合云存储的存储引擎设计构想


> 本文由原作者归档自知乎，原文发布于 2022-07-07。原文链接：[深度结合云存储的存储引擎设计构想](https://zhuanlan.zhihu.com/p/538576387)。

最近，TiKV发布了新的存储引擎去适配Multi-Raft和云存储。https://github.com/tikv/raft-engine。在这个设计中，让我最受启发的是对IO的思考。

我们都知道云存储(云盘存储、云文件存储、对象存储)具有着非常大的存储空间、极高的吞吐和QPS性能，但是每次访问的IO时延却很大。虽然，随着云存储的技术发展，IO时延会不断地降低，但是想要做到本地nvme ssd的性能效果，还是有非常大的挑战。既然IO时延不好降低，索性从存储引擎方面入手，尽量减少IO操作、提高QPS和吞吐也能达到不错的效果。

TiKV的Raft-Engine的确也是这么做的，https://docs.pingcap.com/zh/tidb/stable/release-6.1.0

![图 1](/images/shen-du-jie-he-yun-cun-chu-de-cun-chu-yin-qing-she-ji-gou-xiang.md/img-01.jpg)

在Raft-Engine里面，还有很多其他的技巧去进一步降低IO次数、提高并发等。比如，多个write请求通过write group方式合并到一起去操作，将多个IO请求搞成1个。

未来应该出现一款新的存储引擎，每次write有极少的IO请求、但单次IO的size可以很大、或者有着极高的并行写能力。比如：

- 云存储本身可以保证数据的可靠性，有什么数据直接写就可以，那么存储引擎里面的block size就不在成为必须。此外，很多计算可以通过内存先buffer再批量写，从而提高吞吐，提高效率
- 云存储本身是分布式管理数据的，那么可以将存储引擎按照table或者region级别管理各自的wal，通过wal并行写的方式来提高QPS，从而提高效率
- 既然云存储的IO延迟不好解决，索性减少IO操作。Rocksdb设计比较好的地方就在于，每次update单行数据只有一个写WAL的IO请求，不好的地方是Flush和Compact会在某个时间占用很多的IO，而且Flush和Compact并不能充分利用云存储高QPS的特点，这是未来结合云需要改造的点。
- 或者，不经过linux的kernel直接操作云存储读写数据

如果有一款存储引擎， 能够按照这样的方式去深度结合云存储的现状去设计，那么上层的数据库server设计将会更简单(毕竟云存储可以保证数据可靠不丢失，那么server层都不需要raft三副本了)，这会是有一次颠覆性的产品。

