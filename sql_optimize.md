#SQL优化（1）
##优化SQL的一般步骤
本文所涉及案例表来自MySQL的案例库sakila（官方提供的模拟电影出租厅信息管理系统的数据库），[点击下载][1]，压缩包包含sakila-schema.sql、sakila-data.sql和sakila.mwb分别为表结构，数据和MySQL Workbench模型。

###通过show status了解SQL执行频率
查看服务器状态信息：`show [session|global] status`

>session 代表当前连接，global为自数据库上次启动至今统计结果。

显示当前session中所有统计参数的值：

```
show status like 'Com_%';

+-----------------------------+-------+
| Variable_name               | Value |
+-----------------------------+-------+
| Com_admin_commands          | 0     |
| Com_assign_to_keycache      | 0     |
| Com_alter_db                | 0     |
| Com_alter_db_upgrade        | 0     |
| Com_alter_event             | 0     |
| Com_alter_function          | 0     |
| Com_alter_instance          | 0     |
| Com_alter_procedure         | 0     |
| Com_alter_server            | 0     |
| Com_alter_table             | 2     |
```

Com_xxx表示每个xxx语句执行次数，常关心：Com_select/insert/update/delete，以上对所有存储引擎的表都会累计。针对Innodb引擎，累加算法略有不同，分别为：Innodb_rows_read/inserted/updated/deleted。通过这些参数可以了解当前数据库是以插入更新为主还是查询操作为主，以及各类SQL执行比例。对于事务型应用，可通过Com_commit和Com_rollback了解事务提交和回滚的情况。

####以下参数便于用户了解数据库基本情况：
- Connections:试图连接MySQL服务器的次数
- Uptime:服务器工作时间
- Slow_queries:慢查询次数

###定位执行效率低的SQL语句
- 通过慢查询日志定位执行效率低的SQL语句，用--log-slow-queries[=file_name]选项启动时，超过long_query_time的语句会记录在文件中。
- `show processlist`查看当前MySQL在进行的线程，查看线程状态、是否锁表等执行情况，同时会对一些锁表操作进行优化。 

###通过EXPLAIN分析SQL的执行计划
找到效率低的SQL后，可通过EXPLAIN或者DESC命令获取MySQL如何执行SELECT语句的信息，如执行查询过程中表如何连接和连接顺序等，如：

```
mysql> desc select sum(amount) from customer a,payment b where 1=1 and a.customer_id=b.customer_id and email='JANE.BENNETT@sakilacustomer.org'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: a
   partitions: NULL
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 599
     filtered: 10.00
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: b
   partitions: NULL
         type: ref
possible_keys: idx_fk_customer_id
          key: idx_fk_customer_id
      key_len: 2
          ref: sakila.a.customer_id
         rows: 26
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.01 sec)
```
- select_type：select的类型，常见有SIMPLE（简单表）、PRIMARY（主查询）、UNION、SUBQUERY（子查询中的第一个SELECT）等。
- table：输出结果集的表
- type：在表中找到所需行的方式，也叫访问类型。性能为：ALL(全表)<idnex(全索引)<range(索引范围扫描)<ref(非唯一索引或者唯一索引前缀)<eq_ref(唯一索引)<const,system(单表中最多只有一个匹配行)<NULL(不需访问表或索引)
	
	```
	mysql> explain select * from (select * from customer where email='AARON.SELBY@sakilacustomer.org')a\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: customer
   partitions: NULL
         type: const
possible_keys: uk_email
          key: uk_email
      key_len: 153
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.01 sec)

	```
	
- possible_keys:可能使用的索引
- key：实际使用的索引
- key_len：使用到索引字段的长度
- rows：扫描行的数量
- Extra：执行情况的说明

MySQL4.1引入了`explain extended`，配合`show warnings`可以看到SQL真正被执行之前优化器做了哪些SQL改写。

MySQL5.1支持分区后，explain也对分区增加了支持，通过`explain partition`可查看SQL所访问的分区。
>通过explain还不能定位SQL问题，此时可以选择profile的联合分析

###通过show profile分析SQL
MySQL从5.0.37增加对`show profiles`和`show profile`的支持，通过have_profiling参数可查看是否支持profile。默认profiling是关闭的，可通过set在Session级别开启：

```
mysql> select @@have_profiling;
+------------------+
| @@have_profiling |
+------------------+
| YES              |
+------------------+
1 row in set, 1 warning (0.00 sec)

mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> set profiling=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

通过profile可以更清楚了解SQL执行过程。通过`show profile for query id`可查看执行过程中线程的每个状态和耗时：

```
mysql> show profiles;
+----------+------------+-------------------------------+
| Query_ID | Duration   | Query                         |
+----------+------------+-------------------------------+
|        1 | 0.00013100 | SELECT DATABASE()             |
|        2 | 0.00013600 | SELECT DATABASE()             |
|        3 | 0.00040900 | show databases                |
|        4 | 0.00043100 | show tables                   |
|        5 | 0.00039900 | select count(*) from customer |
+----------+------------+-------------------------------+
5 rows in set, 1 warning (0.00 sec)

mysql> show profile for query 5;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000073 |
| checking permissions | 0.000009 |
| Opening tables       | 0.000020 |
| init                 | 0.000017 |
| System lock          | 0.000009 |
| optimizing           | 0.000192 |
| executing            | 0.000011 |
| end                  | 0.000007 |
| query end            | 0.000012 |
| closing tables       | 0.000010 |
| freeing items        | 0.000020 |
| cleaning up          | 0.000019 |
+----------------------+----------+
12 rows in set, 1 warning (0.00 sec)
```

MySQL支持进一步选择all、cpu、block io、context、switch、page faults等明细类型查看MySQL在使用什么资源上耗费过高时间：

```
mysql> show profile cpu for query 5;
+----------------------+----------+----------+------------+
| Status               | Duration | CPU_user | CPU_system |
+----------------------+----------+----------+------------+
| starting             | 0.000073 | 0.000067 |   0.000006 |
| checking permissions | 0.000009 | 0.000006 |   0.000003 |
| Opening tables       | 0.000020 | 0.000018 |   0.000001 |
| init                 | 0.000017 | 0.000016 |   0.000002 |
| System lock          | 0.000009 | 0.000008 |   0.000001 |
| optimizing           | 0.000192 | 0.000191 |   0.000002 |
| executing            | 0.000011 | 0.000008 |   0.000002 |
| end                  | 0.000007 | 0.000005 |   0.000001 |
| query end            | 0.000012 | 0.000012 |   0.000002 |
| closing tables       | 0.000010 | 0.000008 |   0.000001 |
| freeing items        | 0.000020 | 0.000008 |   0.000011 |
| cleaning up          | 0.000019 | 0.000018 |   0.000002 |
+----------------------+----------+----------+------------+
12 rows in set, 1 warning (0.00 sec)
```

还可通过source查看SQL解析过程中每步源码：

```
mysql> show profile source for query 5;
+----------------------+----------+-----------------------+----------------------+-------------+
| Status               | Duration | Source_function       | Source_file          | Source_line |
+----------------------+----------+-----------------------+----------------------+-------------+
| starting             | 0.000073 | NULL                  | NULL                 |        NULL |
| checking permissions | 0.000009 | check_access          | sql_authorization.cc |         835 |
| Opening tables       | 0.000020 | open_tables           | sql_base.cc          |        5649 |
| init                 | 0.000017 | handle_query          | sql_select.cc        |         121 |
| System lock          | 0.000009 | mysql_lock_tables     | lock.cc              |         323 |
| optimizing           | 0.000192 | optimize              | sql_optimizer.cc     |         151 |
| executing            | 0.000011 | exec                  | sql_executor.cc      |         119 |
| end                  | 0.000007 | handle_query          | sql_select.cc        |         199 |
| query end            | 0.000012 | mysql_execute_command | sql_parse.cc         |        5004 |
| closing tables       | 0.000010 | mysql_execute_command | sql_parse.cc         |        5056 |
| freeing items        | 0.000020 | mysql_parse           | sql_parse.cc         |        5630 |
| cleaning up          | 0.000019 | dispatch_command      | sql_parse.cc         |        1901 |
+----------------------+----------+-----------------------+----------------------+-------------+
```

###通过trace分析优化器如何选择执行计划
MySQL5.6提供了对SQL跟踪trace，通过trace可了解优化器为何选择优化器A而不是B。使用方式：首先打开trace，格式为JSON，设置最大使用内存，避免不够不能完整显示解析过程；然后执行想做trace的SQL，最后检查INFORMATION_SCHEMA.OPTIMIZER_TRACE即可。

```
mysql> select @@optimizer_trace;
+-------------------------+
| @@optimizer_trace       |
+-------------------------+
| enabled=on,one_line=off |
+-------------------------+
1 row in set (0.00 sec)

mysql> set optimizer_trace_max_mem_size=1000000;
Query OK, 0 rows affected (0.00 sec)

mysql> select rental_id from rental where 1=1 and rental_date>='2005-05-25 04:00:00' and rental_date<='2005-05-25 05:00:00' and inventory_id=4466;
+-----------+
| rental_id |
+-----------+
|        39 |
+-----------+
1 row in set (0.01 sec)

mysql> select * from information_schema.optimizer_trace\G
*************************** 1. row ***************************
                            QUERY: select rental_id from rental where 1=1 and rental_date>='2005-05-25 04:00:00' and rental_date<='2005-05-25 05:00:00' and inventory_id=4466
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
        ................
```

##索引问题
索引是数据库优化**最常用**也**是最要**方法之一，本节讨论MySQL索引分类、存储、使用的方法。
###索引存储分类
索引是MySQL存储引擎层中实现的，不是在服务层。故每种引擎所用不一定完全相同，MySQL目前提供4中索引：
- B-Tree索引：常见索引，大部分引擎支持。
- HASH索引：Memory引擎支持。
- R-Tree索引（空间索引）：为MyISAM的一个特殊索引类型，主要用于地理空间数据类型，使用较少。
- Full-text（全文索引）：MyISAM的特殊索引，InnoDB从5.6开始支持。

MySQL不支持函数索引，但可对列前某一部分进行索引，如title的前10个字符，该特性大大缩小了索引文件的大小，但是在排序分组时无法使用。

常用的索引为B-Tree索引和HASH索引，HASH只有Memory/Heap引擎支持，且只适用于Key-Value查询，通过Hash索引更快，但其不适用于范围查找，只有在where条件中是用“=”查找才会使用索引。

###MySQL如何使用索引



[1]: http://downloads.mysql.com/docs/sakila-db.zip

