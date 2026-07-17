# mysql limit offset 原理分析与使用


> 本文由原作者归档自 CSDN，原文发布于 2021-03-16。原文链接：[mysql limit offset 原理分析与使用](https://blog.csdn.net/LIUHUAN0520/article/details/114869738)。

## 背景

一个行数为4亿条的表。

![图 1](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-01.png)

查询50000000~50000010行之间的数据。发现查询时间达到20s！！！

![图 2](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-02.png)

查询执行计划发现，需要进行全表扫描，没有索引。

![图 3](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-03.png)

但是，sbtest1这个表是有索引的

![图 4](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-04.png)

为什么mysql没有选择索引，而是全表扫描呢？

## 分析

### mysql select 语法

```mysql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
    [HIGH_PRIORITY]
    [STRAIGHT_JOIN]
    [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
    [SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
    select_expr [, select_expr] ...
    [into_option]
    [FROM table_references
      [PARTITION partition_list]]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}, ... [WITH ROLLUP]]
    [HAVING where_condition]
    [WINDOW window_name AS (window_spec)
        [, window_name AS (window_spec)] ...]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [into_option]
    [FOR {UPDATE | SHARE}
        [OF tbl_name [, tbl_name] ...]
        [NOWAIT | SKIP LOCKED]
      | LOCK IN SHARE MODE]
    [into_option]

into_option: {
    INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
  | INTO DUMPFILE 'file_name'
  | INTO var_name [, var_name] ...
}
```

[https://dev.mysql.com/doc/refman/8.0/en/select.html](https://dev.mysql.com/doc/refman/8.0/en/select.html)

### offset 实现

MySQL的`limit m n`工作原理就是先读取前面`m+n`条记录，然后抛弃前`m`条，读后面`n`条想要的，所以`m`越大，偏移量越大，性能就越差。 [https://cloud.tencent.com/developer/article/1505252](https://cloud.tencent.com/developer/article/1505252)

## 解决

在网上找到一个解决办法：可以把limit的offset当做where条件，这样mysql直接走索引，通过B+树直接定位到offset位置。类似于看书用书签标记下看到哪里的思路。

![图 5](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-05.png)

查询执行计划发现的确如此

![图 6](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-06.png)

如果使用非聚簇索引，也可以达到相同的效果

![图 7](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-07.png)

## 引申

比如：订单表、商品表，如果想做到分页效果、并提高 用户体验 ，选择这种优化会有很大的提升。

就好比openstack的 开源项目 trove模块就有此设计，通过marker参数来保证一直分页的offset。

marker的使用逻辑为：

- 本次查询的最后一条记录标记为next_marker，
- 将next_marker作为api结果的一部分返回给用户，
- 用户下次再请求这个api时，把next_marker赋值给marker传入进来

### 实现逻辑

- 由于trove支持多种数据库，所以mysql user这类url需要通过注册的方式添加

![图 8](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-08.png)

[https://github.com/openstack/trove/blob/master/trove/extensions/routes/mysql.py#L48](https://github.com/openstack/trove/blob/master/trove/extensions/routes/mysql.py#L48)

- def index方法是/users的controller层实现，在这里调用了models.Users.load(context)方法，第一个参数是context，这个参数是trove封装的上下文里面包含一些基本信息。

![图 9](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-09.png)

[https://github.com/openstack/trove/blob/master/trove/extensions/mysql/service.py#L64](https://github.com/openstack/trove/blob/master/trove/extensions/mysql/service.py#L64)


- 从上面controller层，调用到了models.User.load(context)方法，经过几次转换，调用到client.list_users(marker)方法参数带有marker。调用流程Users.load() -> load_via_context() -> Users.load_with_client(context.marker) -> client.list_users(marker)

![图 10](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-10.png)

[https://github.com/openstack/trove/blob/master/trove/extensions/mysql/models.py#L182](https://github.com/openstack/trove/blob/master/trove/extensions/mysql/models.py#L182)

- client.list_users的实现（manager层）

![图 11](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-11.png)

[https://github.com/openstack/trove/blob/master/trove/guestagent/datastore/manager.py#L833](https://github.com/openstack/trove/blob/master/trove/guestagent/datastore/manager.py#L833)

- list_users的service层实现，注释很详细，就不解释了。

![图 12](/images/mysql-limit-offset-yuan-li-fen-xi-yu-shi-yong.md/img-12.png)

[https://github.com/openstack/trove/blob/master/trove/guestagent/datastore/mysql_common/service.py#L340](https://github.com/openstack/trove/blob/master/trove/guestagent/datastore/mysql_common/service.py#L340)

