#SQL优化
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
- select_type：select的类型，常见有SIMPLE（简单表）

[1]: http://downloads.mysql.com/docs/sakila-db.zip

