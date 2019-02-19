# DBA 五分钟速成

## 1.1 识别性能问题
```md
在确定不是物理资源瓶颈之后，就应该把注意力转向MySQL。
```

### 寻找运行缓慢的SQL语句
```sql
mysql> SHOW FULL PROCESSLIST\G

     Id: 13774265
   User: t_test@sunny
   Host: 127.0.0.1:36994
     db: db_test
Command: Query
   Time: 0
  State: starting
   Info: SHOW FULL PROCESSLIST
```
```md
以上输出中 Time 表示 Info 列出的语句的运行时间。
```

### 确认低效查询
```md
首先确认查询是否每次重复执行都很慢。
即需要验证这个低效查询不是由于锁或系统瓶颈等其它因素导致的个别现象。
```
* 重复运行SQL语句并记录执行时间
```md
注意：
重复执行的方法只适用于 SELECT 语句，因为它不会修改现有数据。
如果是 UPDATE 或者 DELETE，那么可以重写为 SELECT 完成验证。
```

* 生成一个查询执行计划（Query Execution Plan， QEP）
```md
QEP 决定了MySQL 从存储引擎获取数据的方式。
```
```sql
mysql> EXPLAIN SELECT id,status,s_id FROM db_test.t_test WHERE s_id  = 1\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: jobs
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1795
     filtered: 10.00
        Extra: Using where
```
```md
* 输出
key 列 输出 查询使用的 索引，任何没有使用索引的查询都可以被认为是没有足够调优的查询。
rows 列 显式受影响的行数，可以用来估计查询需要读取的数据量，这是和查询时间密切相关的。
type 列 显式 ALL 也会是潜在性能问题的一个标志。（具体参考第4，9章）

* 解析
上例中 key 未显式索引，由于这是一个单表 SELECT 语句，可以理解为对整个表进行了扫描并找到符合条件的行。
结果中的 rows 列 可以被认为是一个近似值。

* 注意
大多数情况下 EXPLAIN 并不实际运行SQL，当优化器需要执行这条SQL 语句的一部分来决定如何构造 QEP 时会例外。

存储引擎不同，Rows 显式的行数可能是一个估计值，也可能是精确值。
不过即使是一个估计值（InnoDB），通常也足以使优化器做出一个有充分依据的决定。
``` 

## 1.2 优化查询
### 不应该做的事情
> 在没有验证的情况下，不能就直接添加索引解决问题。
```md
ALTER 是阻塞操作，在一个表有多个索引的情况下 DML 语句的性能开销可能会更大。
```
### 确认优化
```md
在决定添加索引之前，至少做两项检查：
* 表现有的结构
* 表的大小
```
```sql
mysql> SHOW TABLE STATUS LIKE 't_test'\G
*************************** 1. row ***************************
           Name: t_test
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 1795
 Avg_row_length: 13152
    Data_length: 23609344
Max_data_length: 0
   Index_length: 0
      Data_free: 31457280
 Auto_increment: 2837
    Create_time: 2018-09-17 15:32:44
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options:
        Comment: t_test
1 row in set (0.00 sec)
```
```md
通过 Data_length  和 Rows 信息 来获取 表大小的一个近似值。
```

### 备选解决方案
```md
如果 表 存在相关的联合索引，也可以通过修改 WHERE 子句中的条件，来使得现有的索引被使用。
这样可以不必要改变现有的数据库模式。

有时候添加 索引并不是解决查询慢的理想方法，反而会因为添加了不必要的索引，增加了额外的开销。
```



