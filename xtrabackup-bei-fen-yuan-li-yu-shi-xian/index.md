# xtrabackup备份原理与实现


> 本文由原作者归档自 CSDN，原文发布于 2021-02-04。原文链接：[xtrabackup备份原理与实现](https://blog.csdn.net/LIUHUAN0520/article/details/113653749)。

## 预备知识

xtrabackup备份时，为了保证数据的一致性，会执行以下几个sql来保证位点的准确。

```
200817 10:20:33 2141744 Query SET SESSION lock_wait_timeout=31536000
2141744 Query FLUSH NO_WRITE_TO_BINLOG TABLES
2141744 Query FLUSH TABLES WITH READ LOCK
2141744 Query SHOW MASTER STATUS
2141744 Query SHOW VARIABLES
2141744 Query FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS
2141744 Query UNLOCK TABLES
2141744 Query SELECT UUID()
2141744 Query SELECT VERSION()
```

在这里面会上 FLUSH TABLES WITH READ LOCK。这就要求我们理解下隔离级别和事务之间的关系了。

## update和FTWL关系

基于上面的问题，进行了如下4个实验

![图 1](/images/xtrabackup-bei-fen-yuan-li-yu-shi-xian.md/img-01.png)

查阅资料发现，FTWRL执行，主要包括3个步骤：

- 1.上全局读锁(lock_global_read_lock)
- 2.清理表缓存(close_cached_tables)
- 3.上全局COMMIT锁(make_global_read_lock_block_commit)

更详细的说明，可见连接。[https://www.cnblogs.com/cchust/p/4603599.html](https://www.cnblogs.com/cchust/p/4603599.html)

1. 上全局读锁会导致所有更新操作都会被堵塞；
2. 关闭表过程中，如果有大查询导致关闭表等待，那么所有访问这个表的查询和更新都需要等待；
3. 上全局COMMIT锁时，会堵塞活跃事务提交。

FTWRL中的第1和第3步都是通过MDL锁实现。

## xtrabckup备份流程图

[https://www.cnblogs.com/linuxk/p/9372990.html](https://www.cnblogs.com/linuxk/p/9372990.html)

![图 2](/images/xtrabackup-bei-fen-yuan-li-yu-shi-xian.md/img-02.png)

## 代码实现

### 主函数流程

![图 3](/images/xtrabackup-bei-fen-yuan-li-yu-shi-xian.md/img-03.png)

### main函数

```
int main(int argc, char **argv) {
  //...


  /* --backup */
  if (xtrabackup_backup) {
    xtrabackup_backup_func();
  }

  //...
  msg_ts("completed OK!\n");
}
```

### 真正的代码

主流程

1. 开启redo文件复制线程
2. 根据并发数，开启数据文件复制线程，
3. while(1) 直到数据文件全部复制完成
4. 开始备份非innodb文件，为了获取一致性位点，会 stop slave sql_thread
5. 获取位点信息
6. 收尾工作

```
void xtrabackup_backup_func(void) {
  //....
  init_mysql_environment();
  xb_normalize_init_values();


  //[0] start redo log copy thread
  Redo_Log_Data_Manager redo_mgr;
  redo_mgr.set_copy_interval(xtrabackup_log_copy_interval);
  redo_mgr.init()
  os_thread_create(PFS_NOT_INSTRUMENTED, io_watching_thread).start();
  redo_mgr.start()

  //[1-n] Create data copying threads
  data_threads = (data_thread_ctxt_t *)ut_malloc_nokey(sizeof(data_thread_ctxt_t) * xtrabackup_parallel);
  count = xtrabackup_parallel;
  for (i = 0; i < (uint)xtrabackup_parallel; i++) {
    os_thread_create(PFS_NOT_INSTRUMENTED, data_copy_thread_func, data_threads + i).start();
  }

  //[...] Wait for threads to exit
  while (1) {
    if (count == 0) {break;}
    if (redo_mgr.is_error()) {exit(EXIT_FAILURE);}
  }

  //[...] Backup non-InnoDB data
  Backup_context backup_ctxt;
  backup_start(backup_ctxt)

  //[...] stop redo log copy
  redo_mgr.stop_at(backup_ctxt.log_status.lsn)
  redo_mgr.is_error()
  metadata_to_lsn = redo_mgr.get_last_checkpoint_lsn();
  metadata_last_lsn = redo_mgr.get_stop_lsn();
  xtrabackup_stream_metadata(ds_meta)


  //[...] stop backup
  backup_finish(backup_ctxt)

  //[...] write file
  sprintf(filename, "%s/%s", xtrabackup_extra_lsndir, XTRABACKUP_METADATA_FILENAME);
  xtrabackup_write_metadata(filename)
  sprintf(filename, "%s/%s", xtrabackup_extra_lsndir, XTRABACKUP_INFO);
  xtrabackup_write_info(filename))

  msg("xtrabackup: Transaction log of lsn (" LSN_PF ") to (" LSN_PF ") was copied.\n",redo_mgr.get_start_checkpoint_lsn(), redo_mgr.get_scanned_lsn());
}
```

### 备份非innodb表

备份非事务文件之前会上FTWL锁，然后备份文件，并获取当时的位点。

```
bool backup_start(Backup_context &context) {


  lock_tables_maybe(mysql_connection, opt_backup_lock_timeout, opt_backup_lock_retry_count)


  backup_files(MySQL_datadir_path.path().c_str(), false)


  /* There is no need to stop slave thread before copying non-Innodb data when
  --no-lock option is used because --no-lock option requires that no DDL or
  DML to non-transaction tables can occur. */
  wait_for_safe_slave(mysql_connection)

  xb_mysql_query(mysql_connection, "FLUSH NO_WRITE_TO_BINLOG BINARY LOGS", false);


  context.log_status = log_status_get(mysql_connection);

  write_current_binlog_file(mysql_connection)
  write_slave_info(mysql_connection)
  write_binlog_info(mysql_connection);


  xb_mysql_query(mysql_connection, "FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS", false);

  return (true);
}






/*********************************************************************/ /**
 Function acquires either a backup tables lock, if supported
 by the server, or a global read lock (FLUSH TABLES WITH READ LOCK)
 otherwise. If server does not contain MyISAM tables, no lock will be
 acquired. If slave_info option is specified and slave is not
 using auto_position.
 @returns true if lock acquired */
bool lock_tables_maybe(MYSQL *connection, int timeout, int retry_count) {
  bool force_ftwrl = opt_slave_info && !slave_auto_position &&
                     !(server_flavor == FLAVOR_PERCONA_SERVER);

  if (tables_locked || (opt_lock_ddl_per_table && !force_ftwrl)) {
    return (true);
  }

  if (!have_unsafe_ddl_tables && !force_ftwrl) {
    return (true);
  }

  if (have_backup_locks && !force_ftwrl) {
    return lock_tables_for_backup(connection, timeout, retry_count);
  }

  return lock_tables_ftwrl(connection);
}




/*********************************************************************/ /**
 Function acquires a FLUSH TABLES WITH READ LOCK.
 @returns true if lock acquired */
bool lock_tables_ftwrl(MYSQL *connection) {
  xb_mysql_query(connection, "SET SESSION lock_wait_timeout=31536000", false);
    /* We do first a FLUSH TABLES. If a long update is running, the
    FLUSH TABLES will wait but will not stall the whole mysqld, and
    when the long update is done the FLUSH TABLES WITH READ LOCK
    will start and succeed quickly. So, FLUSH TABLES is to lower
    the probability of a stage where both mysqldump and most client
    connections are stalled. Of course, if a second long update
    starts between the two FLUSHes, we have that bad stall.

    Option lock_wait_timeout serve the same purpose and is not
    compatible with this trick.
    */
  xb_mysql_query(connection, "FLUSH NO_WRITE_TO_BINLOG TABLES", false);
  xb_mysql_query(connection, "FLUSH TABLES WITH READ LOCK", false);
  return (true);
}
```

### 优雅的关闭SQL线程

通过不断的开启和关闭 sql_thread，来保证slave打开的tmp_table为0

```
/*********************************************************************/ /**
 Wait until it's safe to backup a slave.  Returns immediately if
 the host isn't a slave.  Currently there's only one check:
 Slave_open_temp_tables has to be zero.  Dies on timeout. */
bool wait_for_safe_slave(MYSQL *connection) {
  mysql_variable status[] = {
{"Read_Master_Log_Pos", &read_master_log_pos},
                             {"Slave_SQL_Running", &slave_sql_running},
                             {NULL, NULL}};
  read_mysql_variables(connection, "SHOW SLAVE STATUS", status, false);


  if (!(read_master_log_pos && slave_sql_running)) {
    msg("Not checking slave open temp tables for --safe-slave-backup because host is not a slave\n");
    goto cleanup;
  }

  if (strcmp(slave_sql_running, "Yes") == 0) {
    xb_mysql_query(connection, "STOP SLAVE SQL_THREAD", false);
  }

retry:
  open_temp_tables = get_open_temp_tables(connection);
  curr_slave_coordinates = get_slave_coordinates(connection);

  while (open_temp_tables && routine_start_time + timeout > (ssize_t)my_time(MY_WME)) {
    msg_ts("Starting slave SQL thread, waiting %d seconds, then checking Slave_open_temp_tables again (%d seconds of sleep time remaining)...\n", sleep_time, (int)(routine_start_time + timeout - (ssize_t)my_time(MY_WME)));

    xb_mysql_query(connection, "START SLAVE SQL_THREAD", false);
    os_thread_sleep(sleep_time * 1000000);

    curr_slave_coordinates = get_slave_coordinates(connection);
    msg_ts("Slave pos:\n\tprev: %s\n\tcurr: %s\n", prev_slave_coordinates,
           curr_slave_coordinates);
    if (prev_slave_coordinates && curr_slave_coordinates &&
        strcmp(prev_slave_coordinates, curr_slave_coordinates) == 0) {
      msg_ts(
          "Slave pos hasn't moved during wait period, "
          "not stopping the SQL thread.\n");
    } else {
      msg_ts("Stopping SQL thread.\n");
      xb_mysql_query(connection, "STOP SLAVE SQL_THREAD", false);
    }

    open_temp_tables = get_open_temp_tables(connection);
    msg_ts("Slave open temp tables: %d\n", open_temp_tables);
  }

  if (open_temp_tables == 0) {
    /* We are in a race here, slave might open other temp tables
    inbetween last check and stop. So we have to re-check
    and potentially retry after stopping SQL thread. */
    xb_mysql_query(connection, "STOP SLAVE SQL_THREAD", false);
    open_temp_tables = get_open_temp_tables(connection);
    if (open_temp_tables != 0) {
      goto retry;
    }

    msg_ts("Slave is safe to backup.\n");
    goto cleanup;
  }
  if (sql_thread_started) {
    xb_mysql_query(connection, "START SLAVE SQL_THREAD", false);
  } else {
    xb_mysql_query(connection, "STOP SLAVE SQL_THREAD", false);
  }

cleanup:
  free(prev_slave_coordinates);
  free(curr_slave_coordinates);
  free_mysql_variables(status);
  return (result);
}
```

### 判断slave开启临时表的个数

降低死锁概率，需要配合timeout使用

```
# xtrabackup 参数说明
  --safe-slave-backup Stop slave SQL thread and wait to start backup until
                      Slave_open_temp_tables in "SHOW STATUS" is zero. If there
                      are no open temporary tables, the backup will take place,
                      otherwise the SQL thread will be started and stopped
                      until there are no open temporary tables. The backup will
                      fail if Slave_open_temp_tables does not become zero after
                      --safe-slave-backup-timeout seconds. The slave SQL thread
                      will be restarted when the backup finishes.


# 官方参数说明
https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html#statvar_Slave_open_temp_tables
The number of temporary tables that the replica SQL thread currently has open. If the value is greater than zero, it is not safe to shut down the replica
```

xtrabackup 的使用方法，直接 show status like 'slave_open_temp_tables'; 方式查看

```
static int get_open_temp_tables(MYSQL *connection) {
  mysql_variable status[] = {
{"Slave_open_temp_tables", &slave_open_temp_tables}, {NULL, NULL}};
  read_mysql_variables(connection, "SHOW STATUS LIKE 'slave_open_temp_tables'", status, true);
  result = slave_open_temp_tables ? atoi(slave_open_temp_tables) : 0;
  return (result);
}
```

### 获取从库relay的位点

直接在show slave status 获取位点信息

```
static char *get_slave_coordinates(MYSQL *connection) {
  char *relay_log_file = NULL;
  char *exec_log_pos = NULL;
  char *result = NULL;
  mysql_variable slave_coordinates[] = {
      {"Relay_Master_Log_File", &relay_log_file},
      {"Exec_Master_Log_Pos", &exec_log_pos},
      {NULL, NULL}};
  read_mysql_variables(connection, "SHOW SLAVE STATUS", slave_coordinates, false);
  ut_a(asprintf(&result, "%s\\%s", relay_log_file, exec_log_pos));
  free_mysql_variables(slave_coordinates);
  return result;
}
```

备份日志举例

```
[root@MSS-d4251dyr3k log]# cat /tmp/xtrabackup.log
200817 10:20:25 version_check Connecting to MySQL server with DSN 'dbi:mysql:;mysql_read_default_group=xtrabackup;host=127.0.0.1;port=3306;mysql_socket=/export/data/mysql/tmp/mysql.sock' as 'os_admin' (using password: YES).
200817 10:20:25 version_check Connected to MySQL server
200817 10:20:25 version_check Executing a version check against the server...
200817 10:20:25 version_check Done.
200817 10:20:25 Connecting to MySQL server host: 127.0.0.1, user: os_admin, password: set, port: 3306, socket: /export/data/mysql/tmp/mysql.sock
Using server version 5.6.39-log
xtrabackup version 2.4.8 based on MySQL server 5.7.13 Linux (x86_64) (revision id: 97330f7)
xtrabackup: uses posix_fadvise().
xtrabackup: cd to /export/data/mysql/data
xtrabackup: open files limit requested 64000, set to 655350
xtrabackup: using the following InnoDB configuration:
xtrabackup: innodb_data_home_dir = /export/data/mysql/data
xtrabackup: innodb_data_file_path = ibdata1:512M;ibdata2:512M:autoextend
xtrabackup: innodb_log_group_home_dir = /export/data/mysql/data
xtrabackup: innodb_log_files_in_group = 3
xtrabackup: innodb_log_file_size = 1073741824
2020-08-17 10:20:25 0x7f41fb10d740 InnoDB: Using Linux native AIO
xtrabackup: using O_DIRECT
InnoDB: Number of pools: 1
200817 10:20:25 >> log scanned up to (8956176)
xtrabackup: Generating a list of tablespaces
InnoDB: Allocated tablespace ID 1 for mysql/innodb_table_stats, old maximum was 0
200817 10:20:26 [01] Compressing, encrypting and streaming /export/data/mysql/data/ibdata1
200817 10:20:26 >> log scanned up to (8956176)
200817 10:20:27 >> log scanned up to (8956176)
200817 10:20:28 >> log scanned up to (8956176)
200817 10:20:29 [01] ...done
200817 10:20:29 [01] Compressing, encrypting and streaming /export/data/mysql/data/ibdata2
200817 10:20:29 >> log scanned up to (8956176)
200817 10:20:30 >> log scanned up to (8956176)
200817 10:20:31 >> log scanned up to (8956176)
200817 10:20:32 >> log scanned up to (8956176)
200817 10:20:32 [01] ...done
200817 10:20:32 [01] Compressing, encrypting and streaming ./mysql/innodb_table_stats.ibd
200817 10:20:32 [01] ...done
200817 10:20:32 [01] Compressing, encrypting and streaming ./mysql/innodb_index_stats.ibd
200817 10:20:32 [01] ...done
200817 10:20:32 [01] Compressing, encrypting and streaming ./mysql/slave_relay_log_info.ibd
200817 10:20:32 [01] ...done
200817 10:20:32 [01] Compressing, encrypting and streaming ./mysql/slave_master_info.ibd
200817 10:20:32 [01] ...done
200817 10:20:32 [01] Compressing, encrypting and streaming ./mysql/slave_worker_info.ibd
200817 10:20:32 [01] ...done
200817 10:20:32 [01] Compressing, encrypting and streaming ./hcloud/t.ibd
200817 10:20:32 [01] ...done
200817 10:20:33 Executing FLUSH NO_WRITE_TO_BINLOG TABLES...
200817 10:20:33 Executing FLUSH TABLES WITH READ LOCK...
200817 10:20:33 Starting to backup non-InnoDB tables and files
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/db.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/db.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/db.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/user.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/user.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/user.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/func.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/func.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/func.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/plugin.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/plugin.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/plugin.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/servers.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/servers.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/servers.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/tables_priv.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/tables_priv.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/tables_priv.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/columns_priv.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/columns_priv.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/columns_priv.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_topic.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_topic.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_topic.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_category.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_category.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_category.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_relation.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_relation.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_relation.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_keyword.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_keyword.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/help_keyword.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_name.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_name.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_transition.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_transition.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_transition.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_transition_type.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_transition_type.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_transition_type.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_leap_second.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_leap_second.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/time_zone_leap_second.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/proc.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/proc.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/proc.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/procs_priv.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/procs_priv.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/procs_priv.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/general_log.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/general_log.CSM to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/general_log.CSV to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/slow_log.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/slow_log.CSM to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/slow_log.CSV to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/event.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/event.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/event.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/ndb_binlog_index.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/ndb_binlog_index.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/ndb_binlog_index.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/innodb_table_stats.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/innodb_index_stats.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/slave_relay_log_info.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/slave_master_info.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/slave_worker_info.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/proxies_priv.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/proxies_priv.MYI to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./mysql/proxies_priv.MYD to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [00] Compressing, encrypting and streaming <STDOUT>
200817 10:20:33 [00] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/db.opt to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/cond_instances.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_current.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_history.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_history_long.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_summary_by_instance.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_summary_by_host_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_summary_by_user_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_summary_by_account_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_summary_by_thread_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_waits_summary_global_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/file_instances.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/file_summary_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/file_summary_by_instance.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/socket_instances.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/socket_summary_by_instance.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/socket_summary_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/host_cache.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/mutex_instances.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/objects_summary_global_by_type.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/performance_timers.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/rwlock_instances.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/setup_actors.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/setup_consumers.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/setup_instruments.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/setup_objects.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/setup_timers.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/table_io_waits_summary_by_index_usage.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/table_io_waits_summary_by_table.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/table_lock_waits_summary_by_table.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/threads.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_current.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_history.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_history_long.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_summary_by_thread_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_summary_by_host_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_summary_by_user_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_summary_by_account_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_stages_summary_global_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_current.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_history.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_history_long.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_summary_by_thread_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_summary_by_host_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_summary_by_user_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_summary_by_account_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_summary_global_by_event_name.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/hosts.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/users.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/accounts.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/events_statements_summary_by_digest.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/session_connect_attrs.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./performance_schema/session_account_connect_attrs.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./hcloud/db.opt to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 [01] Compressing, encrypting and streaming ./hcloud/t.frm to <STDOUT>
200817 10:20:33 [01] ...done
200817 10:20:33 Finished backing up non-InnoDB tables and files
200817 10:20:33 [00] Compressing, encrypting and streaming <STDOUT>
200817 10:20:33 [00] ...done
200817 10:20:33 Executing FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS...
xtrabackup: The latest check point (for incremental): '8956176'
200817 10:20:33 >> log scanned up to (8956176)
xtrabackup: Stopping log copying thread.

200817 10:20:33 Executing UNLOCK TABLES
200817 10:20:33 All tables unlocked
200817 10:20:33 Backup created in directory '/xtrabackup_backupfiles/'
MySQL binlog position: filename 'mysql-bin.007045', position '484', GTID of the last change '85b2d3ec-ccca-11ea-9f99-fa163e7e7509:1-1195,
8bf03311-ccca-11ea-9f99-fa163e9a7213:1'
200817 10:20:33 [00] Compressing, encrypting and streaming <STDOUT>
200817 10:20:33 [00] ...done
200817 10:20:33 [00] Compressing, encrypting and streaming <STDOUT>
200817 10:20:33 [00] ...done
xtrabackup: Transaction log of lsn (8956176) to (8956176) was copied.
200817 10:20:33 completed OK!
```

