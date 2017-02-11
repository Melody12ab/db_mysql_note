本文主要内容：事务、锁、SQL Mode、**分区**

##事务控制和锁定语句
####LOCK TABLE和UNLOCK TABLE
- LOCK TABLES可锁定用于**当前线程**的表，如果表被*其他线程*锁定，则当前线程会等待知道可以获取所有锁定为止。
- UNLOCK TABLES可以释放当前线程获得的任何锁定。

```
LOCK TABLES
	tab_name [AS alias]{READ [LOCAL]|[LOW_PRIORITY] WRITE}
	[,tbl_name [AS alias] {READ [LOCAL]|[LOW_PRIOEITY] WRITE}]...
	
UNLOCK TABLES
```
>在数据导出时加-x参数可以锁全表

####事务控制
```
START TRANSACTION |BEGIN [WORK]
COMMIT [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
ROLLBACK [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
SET AUTOCOMMIT={0|1} 
```
默认情况下MySQL是自动提交的

- START TRANSACTION或BEGIN开始一项事务
- COMMIT和ROLLBACK用于提交或者回滚事务
- CHAIN和RELEASE字句用来定义事务提交或者回滚之后的操作，CHAIN会立即开启一个新事务，并和刚才的事务具有相同的隔离级别，RELEASE则会断开和客户端的连接
- SET AUTOCOMMIT修改**当前连接**提交方式

>在进行一些敏感操作时，可是开个事务，以防删库跑路的风险。

```
敏感操作请记得在终端操作之前
start transaction;
...
commit / rollback;

```

####分布式事务
MySQL使用分布式事务的应用程序涉及一个或多个资源管理器和一个事务管理器

- 资源管理器（RM）用于提供通向事务资源的途径，数据库服务器就是一个资源管理器。
- 事务管理器（TM）用于协调作为一个分布式事务一部分的事务。

执行分布式事务过程使用两阶段提交
- 第一阶段，所有分支被预备好。即他们被TM告知要准备提交。通常以为着用于管理分支的每个RM会记录对于被稳定保存的分支的行动。
- 第二阶段，TM告知RMs是否要提交或回滚

>如果一个事务资源只由一个事务资源组成（单一分支）,则该资源可以被告知同事进行预备和提交

分布式事务（XA事务）的SQL语法：

```
XA {START|BEGIN} xid [JOIN|RESUME]

xid: gtrid [,bqual[,formatID]]

使事务进入PREPARE状态，两阶段提交的第一个提交阶段
XA END xid [SUSPEND[FOR MIGRATE]]
XA PREPARE xid

提交或者回滚分布式事务，为第二阶段
XA COMMIT xid [ONE PHASE]
XA ROLLBACK xid

返回当前数据库中处于PREPARE状态的分支事务的详细信息
XA RECOVER

eg:
xa start 'test','dept';

insert into dept values(3,'xiaobai');

xa end 'test','dept';

xa prepare 'test','dept';

xa recover \G

xa commit 'test','dept'
```
>启用mysqlbinlog，用于恢复数据用

##SQL中的安全问题
在日常开发中，开发人员一般只关心SQL是否能实现预期的功能，而对于SQL的安全问题一般不太重视。最常见的就是SQL注入的安全威胁。

预防SQL注入措施：
- 使用预编译语句绑定变量
- 使用应用程序提供的转换函数
- 自己定义函数进行校验
	- 整理数据使之有效
	- 拒绝已知的非法数据
	- 只接受已知的合法输入

##SQL Model及相关问题
SQL Mode定义了MySQL应支持的SQL语法、数据校验等，这样可以更容易地在不同的环境中使用MySQL。

在MySQL中，SQL Model常用来解决以下问题：
- 通过设置SQL Model，可以完成不同严格程度的数据校验，有效地保障数据准确性
- 通过设置SQL Model为ANSI模式，来保证大多数SQL符合标准的SQL语法，这样应用在不同数据库之间进行迁移时，不需要对业务SQL进行较大修改
- 不同数据库间迁移，通过设置SQL Mode可以使MySQL上的数据更方便地迁移到目标数据库中

查看SQL Mode：`select @@sql_mode`

修改sql_mode：` SET [SESSION|GLOBAL] sql_mode='modes' ` 

启动时指定：`--sql-mode="modes"`

####SQL Mode的常见功能
- 检验日期数据合法性，在ANSI模式下，非法日期可以插入，但值为"0000-00-00 00:00:00",系统提示warning，在TRADITIONAL模式下，会拒绝插入并报错。
- 在INSERT或UPDATE过程中，如果SQL Mode处于TRADITIONAL严格模式，运行MOD(x,0)会产生错误，而在非严格该模式返回NULL
- 启用NO_BACKSLASH_ESCAPES模式，使反斜线成为普通字符，在导入数据时或许会用到。
- 启用PIPES_AS_CONCAT模式，将“||”视为字符串连接操作符，在Oracle数据库中“||”被作为连接操作符，在其他数据库中则无法执行。

####常见的SQL Mode
MySQL的SQL Mode（具体版本建议参考相关版本官方网文档）

sql_mode值|描述
---|---
ANSI|等同于REAL_AS_FLOAT、PIPES_AS_CONCAT、ANSI_QUOTES、IGNORE_SPACE和ANSI组合模式，该模式使语法和行为更符合标准的SQL
STRICT_TRANS_TABLES|适用于事务表和非事务表，它是严格模式，不允许非法日期，也不允许超过字段长度的值插入字段中，对于插入不正确的值给出错误而不是警告
TRADITIONAL|等同于STRICT_TRANS_TABLES、STRICT_ALL_TABLES、NO_ZERO_IN_DATE、NO_ZERO_DATE、ERROR_FOR_DIVISION_BY_ZERO、TRANDITIONAL和NO_AUTO_CREATE_USER组合模式，也为严格模式。可应用在事务表和非事务表，用于事务表时，只要出现错误就会立即回滚

####SQL Mode在迁移中如何使用
MySQL提供了很多数据库的组合模式名称，在异构数据库之间迁移数据时可以尝试使用这些模式来导出适合于目标数据库格式的数据

组合后的模式名称|组合中的各个sql_mode
---| ---
DB2| PIPLES_AS_CONCAT 、ANSI_QUOTES、IGNORE_SPACE、NO_KEY_OPTIONS、NO_TABLE_OPTIONS、NO_FIELD_OPTIONS
MAXDB|PIPES_AS_CONCAT、ANSI_QUOTES、IGNORE_SPACE、NO_KEY_OPTIONS、NO_TABLE_OPTIONS、NO_FIELD_OPTIONS、NO_AUTO_CREATE_USER
MSSQL| PIPES_AS_CONCAT、ANSI_QUOTES、IGNORE_SPACE、NO_KEY_OPTIONS、NO_TABLE_OPTIONS、NO_FIELD_OPTIONS、NO_AUTO_CREATE_USER
ORACLE| PIPES_AS_CONCAT、ANSI_QUOTES、IGNORE_SPACE、NO_KEY_OPTIONS、NO_TABLE_OPTIONS、NO_FIELD_OPTIONS、NO_AUTO_CREATE_USER、NO_AUTO_CREATE_USER
POSTGRESQL| PIPES_AS_CONCAT、ANSI_QUOTES、IGNORE_SPACE、NO_KEY_OPTIONS、NO_TABLE_OPTIONS、NO_FIELD_OPTIONS

>SQL Mode的严格模式为MySQL提供了很好的数据校验功能，保证了数据的准确性，TRANDITIONAL和STRICT_TRANS_TABLES是常用的两种严格模式。

##MySQL分区
MySQL从5.1开始支持分区。分区指根据一定规则，数据库把一个表分解成多个更小更容易管理的部分。就数据库的应用而言，逻辑上只有一个表或一个索引，但实际该表可能由多个物理分区对象组成，每个分区都是一个独立对象，可以独自处理，可以作为表的一部分处理。

MySQL分区优点：
- 和单磁盘或文件系统分区比，可以存储更多数据
- 优化查询，在Where字句中包含分区条件时，可以只扫描必要的一个或多个分区；同时在涉及SUM()和COUNT()这类聚合函数的查询时，可以容易地在每个分区上并行处理。
- 对已过期或不需要的数据，可以通过删除分区来快速删除
- 跨磁盘分散数据查询，获得更大的查询吞吐量

####分区概述
