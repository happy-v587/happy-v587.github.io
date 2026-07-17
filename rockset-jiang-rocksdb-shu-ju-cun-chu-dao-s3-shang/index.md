# Rockset：将 RocksDB 数据存储到S3上


> 本文由原作者归档自知乎，原文发布于 2022-12-10。原文链接：[Rockset：将 RocksDB 数据存储到S3上](https://zhuanlan.zhihu.com/p/590773344)。

本文要介绍的是基于RocksDB改造的rocksdb-cloud项目。

首先，可以看下官方的example，这个example展示了如何读写数据，以及如何在无感知的情况下将SST文件上传到S3上。

本人对这段代码进行了debug，并进行深入研究，上传到S3的奥秘如下

![图 1](/images/rockset-jiang-rocksdb-shu-ju-cun-chu-dao-s3-shang.md/img-01.jpg)

实现的方法很巧妙，在执行Mem Table Flush的时候，先将文件写到本地，等RocksDB调用Close方法时再将这个文件Copy到S3上，然后删除本地文件。

通过这个方法，使得上传到S3的流程很简单，对主流程侵入很小；由于内容先写本地后写S3，所以整个SST的落盘IO会比直接写S3性能会好很多，可以保证读写数据的稳定。

