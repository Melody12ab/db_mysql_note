#SQL优化（2）
在上一节介绍了通过索引来优化查询，开发中除了查询外，还有insert，group by等，本章就这些SQL语句介绍一些优化方法。
##大批量插入数据
```
alter table tbl_name DISABLE KEYS;
loading the data
alter table tbl_name ENABLE KEYS;
```

`DISABLE/ENABLE KEYS`用来打开或关闭MyISAM表非唯一索引的更新。在导入大量数据到空MyISAM表，默认是先导入数据后创建索引，这时可以不用设置：

```
mysql> load data infile '/home/mysql/file_test.txt' into table file_test;
50+w数据耗时：100+s

alter table file_test disable keys;
load data...
alter ... enable keys;
同样50+w数据耗时：10+s
```

上面是对MyISAM表导入的优化措施，对InnoDB表:

- 由于是按照主键的顺序保存的，所以将导入的数据**按照主键的顺序排列**可提高效率。
- 导入前`SET UNIQUE_CHECKS=0`，关闭唯一校验，导入结束后恢复。
- 如果打开自动提交建议导入前关闭，即`SET AUTOCOMMIT=0`，导入后恢复。

## 优化INSERT语句
- 同时从同一客户端插入很多行，尽量使用多个值表的INSERT语句:`insert into test values(1,2),(1,3),(1,4)...`
- 从不同客户插入很多行，通过使用`InSERT DELAYED`得到更高速度。DELAYED让INSERT语句马上执行，数据放在了内存的队列中，没有落到磁盘上；`INSERT LOW_PRIORITY`则相反，在其他用户对表读写完成后才插入。
- 索引文件和数据文件分在不同的磁盘上存放（建表选项）。
- 从文件导入数据时，使用`LOAD DATA INFILE`。

## 优化ORDER BY语句
首先了解MySQL中的排序方式：

1. 通过有序索引，顺序扫描直接返回有序数据，在使用explain分析查询时显示Using Index，不需要额外排序，操作效率较高。
2. 通过对返回数据排序，即FileSort排序，所有不是通过索引直接返回排序结果的排序都叫Filesort排序。FileSort不代表通过磁盘文件排序，只是说明进行了一个排序操作，是否使用磁盘会临时表，取决于MySQL服务器对排序参数的设置和需排序的数据的大小。FileSort通过相应排序算法，将取得数据在sort_buffer_size系统变量设置的内存排序区中进行排序，然后将各个快合并成有序结果集。sort_buffer_size设置的排序区是线程独占的，因此同一时刻有多个sort buffer排序区。

以上即为MySQL排序的方式，优化目标：尽量减少额外的排序，通过索引直接返回有序数据。

### 使用索引优化
下列情况使用索引：

```
select * from tbl_name order by key_part1,key_part2...;
select * from tbl_name where key_part1=1 order by key_part1 desc,key_part2 desc;
select * from tbl_name order by key_part1 desc,key_part2 desc;
```

以下情况不使用索引：

```
select * from tbl_name order by key_part1 desc ,key_part2 asc;
--order by 的字段混合ASC和DESC
select * from tbl_name where key2=constant order by key1;
--用于查询行的关键字于order by 中使用的不同
select * from tbl_name oreder by key1,key2;
--对不用的关键字使用order by;
```

### Filesort优化
某些情况下，条件限制不能让Filesort小时，那就需要想办法加快Filesort的操作，对Filesort，MySQL有两种排序算法。

- **两次扫描算法**(Two Passws):先根据条件去除排序字段和行指针信息，之后在排序区sort buffer中排序。如果sort buffer不够，则在临时表中存储排序结果，完成后根据行指针回表读取记录。缺点是需要两次回表读取，可能导致大量随机I/O操作；优点是排序的时候内存开销较少。
- **一次扫描算法**(Single Pass):一次取出所有满足条件的字段，在排序去sort buffer中排序后输出结果集。排序时内存开销较大，但是效率高。

MySQL比较系统变量`max_length_for_sort_data`的大小和Query语句取出的字段总大小来判断使用哪种排序算法。适当的加大`max_length_for_sort_data`的值，能让MySQL选择更优化的FileSort排序算法，过大则会I/O过高；CPU和I/O利用平衡就足够了。适当加大`sort_buffer_size`排序区，尽量让排序在内存中完成，可避免创建零食表在文件中进行。由于`sort_buffer_size`是线程独占的，设置过大会导致服务器swap严重。另外尽量使用必要的字段，避免`select * ...`，

## 优化GROUP BY语句
默认情况下,MySQL会对所有group by后的字段进行排序，这与在查询中指定order by 类似。如果查询包括group by ，但想要避免排序结果的消耗，可指定order by null禁止排序。（新版的MySQL已经修复了这个问题）
> 有时候啊，吭哧吭哧优化了半天，不如直接升级到新版或者直接上SSD。^_^.

## 优化嵌套查询
# 