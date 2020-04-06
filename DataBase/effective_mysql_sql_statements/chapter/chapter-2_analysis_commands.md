# 基本的分析命令

> 示例表
* [mysql.user](infer/table_mysql.user.md)

## EXPLAIN
```md
EXPLAIN 用于确定一条SQL的QEP。

通过 EXPLAIN 可以获取：
很多可能被优化器使用到的访问策略
运行SQL是哪种策略预计会被优化器采用

* 注意
生成的 QEP 可能会因为很多因素发生改变。
MySQL 不会将一个 QEP 和某个给定查询绑定，QEP 会由SQL 每次执行时的实际情况而定，
即使使用存储过程也是如此，尽管存储过程都是运行解析过的。
```
> 不同 WHERE 条件，对 QEP 的影响
```sql
mysql> EXPLAIN SELECT host,user FROM mysql.user where user LIKE 'r%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: index
possible_keys: NULL
          key: PRIMARY
      key_len: 276
          ref: NULL
         rows: 11
     filtered: 11.11
        Extra: Using where; Using index
```
```sql
mysql> EXPLAIN SELECT host,user FROM mysql.user where host='localhost' AND user LIKE 'r%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 276
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using where; Using index
```
> 目前重点可以关注：
```md
* 没有使用索引（key 列 为NULL）
* 很多处理过的行（rows 列）
* 很多被评估的索引（possible_keys 列）
```

* EXPLAIN PARTITIONS
```md
EXPLAIN 的 PARTITIONS 关键字出现在 MySQL 5.1 之后，
用于满足在 partitions 列中的查询的特性表分区提供附加信息。
```

* EXPLAIN EXTENDED
```md
MySQL 优化器可能会在运行时简化或重写 用户的SQL，EXPLAIN EXTENDED 可以深入理解实际执行的SQL 语句。
在个别情况下，EXTENDED 关键字还可以解释：为什么一个显然存在的索引最后却没有被使用。
```
```sql
mysql> SHOW WARNINGS\G
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `db_test`.`t_test`.`id` AS `id`,`db_test`.`t_test`.`status` AS `status`,`db_test`.`t_test`.`s_id` AS `s_id` from `db_test`.`t_test` where (`db_test`.`t_test`.`s_id` = 1)
```

> EXPLAIN 更详细的讲解可以查看 《高性能MySQL》

## SHOW
* SHOW CREATE TABLE
> [示例](infer/table_mysql.user.md)

```md
MySQL dump 工具能够快速 生成用户模式或数据库实例中所有表的定义。
$ mysqldump  -hlocalhost -uhive -phivemetadata -P3306 --no-data hive > schema.sql

还可以 从 [INFORMATION_SCHEMA](infer/information_schema.md) 库的相关表中获取表结构的组件。
```

* SHOW INDEXS
```sql
mysql> SHOW INDEXES FROM t_test\G
*************************** 1. row ***************************
        Table: t_test
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 1795
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
```
```md
信息包括索引的类型和当前报告的MySQL 索引的基数。
该信息也可以从 INFORMATION_SCHEMA.STATISTICS 中获取。

* Cardinality 列
表示索引中每一列的唯一值的估计值。
```
* SHOW TABLE STATUS
```sql
mysql> SHOW TABLE STATUS LIKE  't_test'\G
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
```
```md
查看数据库表的底层大小以及表结构，包括存储引擎类型、版本、数据和索引大小、行的平均长度以及行数。
该信息也可以从 INFORMATION_SCHEMA.TABLES 中获取。
```
* SHOW STATUS
```md
* 语法
SHOW [GLOBAL | SESSION] STATUS，默认是 SESSION。

查看MySQL 服务器的当前内部状态信息，可以从全局角度确定服务器负载的各项指标。
该信息也可以从 INFORMATION_SCHEMA.GLOBAL_STATUS 和 INFORMATION_SCHEMA.SESSION_STATUS 中获取。
```
```sql
mysql> SHOW GLOBAL STATUS LIKE  'Created_tmp_%tables';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Created_tmp_disk_tables | 24477   |
| Created_tmp_tables      | 1769782 |
+-------------------------+---------+
```
```sql
mysql> FLUSH STATUS;

mysql> SELECT id,status,s_id FROM db_test.t_test WHERE s_id  = 1\G
*************************** 1. row ***************************
             id: 2139
         status: 5
           s_id: 1

mysql> SHOW SESSION STATUS LIKE  'handler_read%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Handler_read_first    | 1     |
| Handler_read_key      | 1     |
| Handler_read_last     | 0     |
| Handler_read_next     | 0     |
| Handler_read_prev     | 0     |
| Handler_read_rnd      | 0     |
| Handler_read_rnd_next | 2837  |
+-----------------------+-------+
```
> 书中对 HANDLER_READ_* 描述不够详细，可以参考
> * [THE HANDLER_READ_* STATUS VARIABLES](https://www.fromdual.com/mysql-handler-read-status-variables)

* SHOW VARIABLES
```md
* 语法 
SHOW [GLOBAL | SESSION]  VARIABLE， 默认是 SESSION。
该信息也可以从 INFORMATION_SCHEMA.GLOBAL_VARIABLES 和 INFORMATION_SCHEMA.SESSION_VARIABLES 中获取。
```
```sql
* tmp_table_size
限制 内部创建的临时表的最大内存使用量。

mysql> SHOW SESSION VARIABLES LIKE  'tmp_table_size%';
+----------------+----------+
| Variable_name  | Value    |
+----------------+----------+
| tmp_table_size | 16777216 |
+----------------+----------+
```
## INFORMATION_SCHEMA
```md
INFORMATION_SCHEMA 为SHOW 命令提供了 ANSI SQL 语句接口（实际执行的是SELECT）。

* 注意
对于一个包含很多表并且表都很大的模式，对 INFORMATION_SCHEMA 中表的查询可能需要执行很长时间。
```