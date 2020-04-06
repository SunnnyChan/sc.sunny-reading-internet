# 覆盖索引（Covering Indexes）
```md
如果一个索引包含（或者说覆盖）所有需要查询的字段的值，就称为“覆盖索引”。
覆盖索引能极大的提高性能。
```

* 好处
```md
1. 索引的条目通常远小于数据行大小，所以如果只需要读取索引行，会极大的减少数据访问量。
  这对缓存的负载很重要，这种情况下响应时间大部分花在了数据拷贝上。
  对于IO密集型应用也有帮助，索引比数据更小，更容易全部放入内存
  （对于MyISAM尤其有用，MyISAM能压缩索引以变得更小）。
2. 索引按序存储（至少单个页内如此），对于IO密集型范围查询会比随机从磁盘读取每一行数据的IO更少。
3. 一些存储引擎如MyISAM在内存中只缓存索引，数据则依赖于OS的缓存，因此要访问数据需要一次系统调用。
  这可能会有严重的性能问题，尤其是那些系统滴啊用占了数据访问中的最大开销的场景。
4. 由于InnoDB的聚簇索引，覆盖索引对 InnoDB 特别有用。
  二级索引保存的主键值，如果二级主键能覆盖查询，则可以避免对主键索引的二次查询。
```

* 实现
```md
覆盖索引必要存储索引列的值，而哈希索引、空间索引和全文索引都不存储索引列的值，
所以MySQL只能以B-Tree 索引作为覆盖索引。

另外存储引擎实现覆盖索引的方式也不一样，而且不是所有的存储引擎都支持覆盖索引。
```

* Using index
```md
当发起一个被索引覆盖的查询时（也叫 索引覆盖查询），EXPLAIN的Extra类可以看到“Using index”输出。
例如：一个表有个一多列索引，而MySQL如果只访问者两列，就可以使用这个索引做覆盖索引。
```

* 陷阱
```md
索引覆盖查询可能会导致无法实现优化。

优化器会在执行查询前判断是否有一个索引能进行覆盖。
假设索引覆盖了WHERE条件中的字段，但不是整个查询涉及的字段。
5.5及之前版本不会使用覆盖索引，虽然理论上MySQL可以通过覆盖索引过滤一部分数据，然后再读取需要的数据。

mysql> EXPLAIN SELECT * FROM products WHERE actor='SEAN CARREY' AND title like '%APOLLO%'\G
     *************************** 1. row ***************************
                id: 1
       select_type: SIMPLE
             table: products
              type: ref
     possible_keys: ACTOR,IX_PROD_ACTOR
               key: ACTOR
           key_len: 52


以上查询中 MySQL 由于不能在索引中执行LIKE 操作（底层存储引擎API的限制），
5.5版本级之前只允许在索引中做简单比较操作。
MySQL能子啊索引中做最左前缀匹配的LIKE比较，因为可以转换为简单比较操作，
但如果是通配符开头的LIKE，就无法匹配了。
这种情况 MySQL 只能提取数据行的值 而不是 索引值 来做比较。
```

* 延迟关联
```md
如何解决上述查询的问题？采用一种称为延迟关联的策略。

首先将索引扩展至覆盖单个数据列（artist,title, prod_id），然后重写SQL：
```
```sql
mysql> EXPLAIN SELECT * -> FROM products
JOIN (
   SELECT prod_id
   FROM products
   WHERE actor='SEAN CARREY' AND title LIKE '%APOLLO%'
) AS t1 ON (t1.prod_id=products.prod_id)\G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
               ...omitted...
*************************** 2. row ***************************
           id: 1
  select_type: PRIMARY
        table: products
               ...omitted...
*************************** 3. row ***************************
           id: 2
  select_type: DERIVED
        table: products
         type: ref
possible_keys: ACTOR,ACTOR_2,IX_PROD_ACTOR
          key: ACTOR_2
      key_len: 52
ref: rows: 11
Extra: Using where; Using index
```
```md
因为延迟了对列的访问，所以称为“延迟关联”。
在查询第一个阶段可以使用覆盖索引，在FROM子句中的子查询中找到匹配的prod_id。

虽然无法完全使用覆盖查询，但总算比完全无法利用索引覆盖要好。

这样优化的结果取决于WHERE条件匹配返回的行数。
```

* 小结
```md
大多数存储引擎中，覆盖索引只能覆盖那些值访问索引中部分列的查询。

InnoDB的二级索引的叶子节点都包含了主键的值，
这意味着可以有效利用这些“额外”的主键列来覆盖查询。
```
```sql
mysql> EXPLAIN SELECT actor_id, last_name
-> FROM sakila.actor WHERE last_name = 'HOPPER'\G
     *************************** 1. row ***************************
                id: 1
       select_type: SIMPLE
             table: actor
              type: ref
     possible_keys: idx_actor_last_name
               key: idx_actor_last_name
           key_len: 137
               ref: const
              rows: 2
Extra: Using where; Using index
```
```md
上表中 last_name 字段有二级索引，虽然该索引不包含主键 actor_id，但也能够用于对actor_id做覆盖查询。
```

* 未来MySQL版本的改进
```md
目前的很多限制 是由于 存储引擎API设计导致的，目前的API设计MySQL将过滤条件发送到存储引擎层。
如果后续版本能够做到这点，则可以把查询发送到数据上，而不是像现在这样只能把数据从存储引擎拉到服务器层，
在做过滤。

5.6版本包含了存储引擎API上一个重要改进，称为“索引条件推送（index condition pushdown）”
该特性对查询方式有很大的改善，上面的很多技就不再需要了。
```