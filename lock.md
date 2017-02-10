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

修改sql_mode：`SET [SESSION|GLOBAL] sql_mode='modes'`

启动时指定：`--sql-mode="modes"`


