# MaxScale不支持MySQL8.0的问题


> 本文由原作者归档自 CSDN，原文发布于 2022-01-19。原文链接：[MaxScale不支持MySQL8.0的问题](https://blog.csdn.net/LIUHUAN0520/article/details/122576280)。

## 背景

mysql的版本是8.0.12，通过maxsclae进行读写分离时，业务端使用mysql-connctor-java-8.0.15.jar访问一直报错。

## 现象

使用mysql-connctor-java-8.0.15.jar连接mysql 读写分离时，报错，内容如下:

![图 1](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-01.png)

使用mysql-cli直连mysql不报错，但是版本号有问题

![图 2](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-02.png)

使用mariadb-cli直连mysql，版本号没问题

![图 3](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-03.png)

## 排查

首先想到的是抓包分析，使用代码访问读写分离时，发现server返回给client的版本是5.5.5-8.0.13

![图 4](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-04.png)

然后debug 代码，发现驱动按照5.5.5版本处理的，所以走了如下代码

![图 5](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-05.png)

然后client跟server 初始化 的时候，发了如下sql

![图 6](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-06.png)

```sql
/* mysql-connector-java-8.0.15 (Revision: 79a4336f140499bd22dd07f02b708e163844e3d5) */SELECT  @@session.auto_increment_increment AS auto_increment_increment, @@character_set_client AS character_set_client, @@character_set_connection AS character_set_connection, @@character_set_results AS character_set_results, @@character_set_server AS character_set_server, @@collation_server AS collation_server, @@collation_connection AS collation_connection, @@init_connect AS init_connect, @@interactive_timeout AS interactive_timeout, @@license AS license, @@lower_case_table_names AS lower_case_table_names, @@max_allowed_packet AS max_allowed_packet, @@net_write_timeout AS net_write_timeout, @@query_cache_size AS query_cache_size, @@query_cache_type AS query_cache_type, @@sql_mode AS sql_mode, @@system_time_zone AS system_time_zone, @@time_zone AS time_zone, @@tx_isolation AS transaction_isolation, @@wait_timeout AS wait_timeout
```

因此代码层面的报错，就只这个版本号返回不对导致的了。

## 解决

### 代码

在maxscale仓库里面搜索5.5.5-关键字，发现有两个地方出现，并配有注释说明：

![图 7](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-07.png)

![图 8](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-08.png)

于是查询mariadb-cli代码，发现cli检测到5.5.5-开头会自己去掉，这就解释通了，mariadb自己把生态做了统一，但是没有兼容mysql8和mysql-cli

[MariaDB `mysql_com.h`](https://github.com/MariaDB/server/blob/57f14eab20ae2733eb341f3d293515a10a40bc48/include/mysql_com.h#L44) 和 [MariaDB `client.c`](https://github.com/MariaDB/server/blob/76f4a78ba2639b5abd01a88b24a3c509c11530ce/sql-common/client.c#L3070) 中都能看到这段兼容逻辑。思考了一下官方之所以这么做是因为mariadb10以后的版本和mariadb5版本没有什么变化，为了保证 数据库连接池 或者jddc这种启动的兼容，变相把版本从10.x降低成5.5.5-xxx。但是mysql8也就跟着出现了问题，所以mysql8可以特殊处理下，毕竟先在还没看到mariadb有8这个版本。

修复后的代码如下，先提给官方看看，然后官方给merge了。[MaxScale PR #230：support mysql8.0.x](https://github.com/mariadb-corporation/MaxScale/pull/230)

### 修复后第一个发行版

[MaxScale 2.5.14 release](https://github.com/mariadb-corporation/MaxScale/releases/tag/maxscale-2.5.14)

### 发行版说明

[MariaDB MaxScale 2.5.14 Release Notes](https://mariadb.com/kb/en/mariadb-maxscale-25-mariadb-maxscale-2514-release-notes-2021-07-21/)

![图 9](/images/maxscale-bu-zhi-chi-mysql8-0-de-wen-ti.md/img-09.png)

可以看到最后一个红框，修复了这个bug

