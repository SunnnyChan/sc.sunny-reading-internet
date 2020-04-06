# 索引和锁 （Indexes and Locking）
* 索引可以让查询锁定更少的行
```md 
如果你的查询不访问那些不需要的行，就会锁定更少的行，从两个方面来看对性能都有好处。

首先，虽然 InnoDB 行锁的效率很高，内存使用也很，但是锁定行时仍然会带来额外的开销，
其次，索引超过需要的行会增加锁的竞争并减少并发性。

InnoDB 只有在访问行的时候才加锁，而索引能够减少 InnoDB 访问的行数。
但只有当 InnoDB 在存储引擎层能够过滤掉所有不需要的行时才有效。
如果 无法过滤掉 无效的行，那么 InnoDB 检索到数据并返回给服务器层后，MySQL服务器才能应用WHERE 子句。
这时已经无法避免锁定行了：InnoDB 已经锁住了这些行，适当时候才释放。
在5.1版本后，InnoDB 可以再服务器端过滤掉行后就释放锁，但是在早期版本中 InnoDB 只有在事务提交之后才能释放锁。
```
* 示例
```md
mysql> SET AUTOCOMMIT=0;
mysql> BEGIN;
mysql> SELECT actor_id FROM sakila.actor WHERE actor_id < 5
-> AND actor_id <> 1 FOR UPDATE; 
+----------+
| actor_id | 
+----------+ 
|         2| 
|         3| 
|         4| 
+----------+

查询仅仅返回 2~4 之间的行，但实际上获取了 1~4 之间的行的排它锁。
锁住第1行，是应为查询执行计划选择的是索引范围扫描：
mysql> EXPLAIN SELECT actor_id FROM sakila.actor
-> WHERE actor_id < 5 AND actor_id <> 1 FOR UPDATE;
     +----+-------------+-------+-------+---------+--------------------------+
     | id | select_type | table | type  | key     | Extra                    |
     +----+-------------+-------+-------+---------+--------------------------+
     |  1 | SIMPLE      | actor | range | PRIMARY | Using where; Using index |
     +----+-------------+-------+-------+---------+--------------------------+
即“索引从开头开始获取满足条件 actor_id < 5 的记录。”
服务器并没有告诉 InnoDB 可以过滤第1行的WHERE条件。
Extra 列出现 Using where，这表示服务器将存储引擎返回行以后再应用过滤条件。

如何证明 第1行被锁定?
可以在上面事务未提交的情况下，再创建一个事务尝试锁定第1行：
mysql> SET AUTOCOMMIT=0;
mysql> BEGIN;
mysql> SELECT actor_id FROM sakila.actor WHERE actor_id = 1 FOR UPDATE;
这个查询会挂起，直到第一个事务释放第1行的锁。
这个行为对于基于语句的复制（参考 Chapter 10）的正常运行来说是必要的。
```
* InnoDB 索引和锁的细节
```md
InnoDB 在二级索引上使用共享（读）锁，但访问主键索引需要排它（写）锁。
这消除了覆盖索引的可能性，并且使得 SELECT FOR UPDATE 比 LOCK IN SHARE MODE 或非锁定查询要慢很多。
```
* 总结
```md
即使使用了索引，InnoDB 也可能锁住一些不需要的数据。
如果不能使用索引查找和锁定行的话，可能会更糟糕，MySQL会做全表扫描并锁住所有的行，而不管是不是需要。
```