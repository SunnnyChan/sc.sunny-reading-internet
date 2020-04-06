# 创建高性能的索引
```md
索引是存储引擎用于快速找到记录的一种数据结构，这是索引的基本功能。

* 索引对于良好的性能非常关键
  尤其当数据量越来越大时，索引对性能的影响愈发重要。
  不恰当的索引，在数据量逐渐增大时，性能会急剧下降。
```

### 索引类型
#### B-树索引
```md
存储引擎以不同的方式使用B-Tree索引，性能也各不相同。
B-Tree 意味着所有的值都是按顺序存储的，并且每一个叶子也到根的距离是相等的。
叶子节点比较特别，它们的指针指向的是被索引的数据，而不是其他的节点页。
树的深度与表的大小直接相关。

B-Tree索引列式顺序组织存储的，所以很适合范围查找。
例如一个基于文本域的索引树上，按字母顺序传递连续的值进行查找是非常合适的，
所以找出所有以I到K开头的名字，会非常高效。

* 注意
对多个列值进行排序，依据表创建语句中定义索引时列的顺序。
```
```sql
CREATE TABLE People (
   last_name  varchar(50) not null,
   first_name varchar(50) not null,
   dob        date not null,
   gender     enum('m', 'f') not null,
   key(last_name, first_name, dob)
);
```
* 使用 B-Tree 索引的查询类型
```md
B-Tree 索引 适合全键值、键值范围和键前缀查找。其中键前缀查找只适用于最左前缀的查找。 
* 全值匹配
  指的是和索引中的所有列进行匹配。
* 匹配最左前缀
  上述表中的所有可以用于朝赵所有姓为Allen的人，即只使用索引的第一列。
* 匹配列前缀
  也可以只匹配某一列的值的开头部分。如查找所有以J开头的姓的人。
* 匹配范围值
  索引可以用于查找姓在Allen 和 Barrymore之间的人。
* 精确匹配某一列并范围匹配另外一列
  如查找 所有型为 Allen，并且名字是字母K开头的人。
  即第一列last_name 全匹配，第二列 first_name 范围匹配。
* 只访问索引的查询
  B-Tree通常支持“只访问索引的查询”，即查询只需要访问索引，不需要访问表。
* 其他
  索引树中的节点是有序的，所以还可以用于查询中的ORDER BY 操作（按顺序查找）。
  一般来说，如果B-Tree可以按照某种方式查找值，那也可以按照这种方式用于排序。
  所以如果 ORDER BY 子句满足前面列出的几种查询类型，则这个索引也可以满足对应的排序需求。
```
* B-Tree 索引的限制
```md
1. 如果不是按照索引的最左列开始查找，则无法使用索引。
  如无法用于查找名字为Bill的人，也无法查找某个特定生日的人。
  类似也无法查找以某个字母结尾的人。
2. 不能跳过索引中的列
  如无法用于查找姓为Smith 并且在某个特定日期出生的人。
  如果不指定名，则只能使用索引的第一列。
3. 如果查询中有某个列的范围查询，则其右边所有列都无法使用索引优化查询
  例如查询 WHERE last_name='Smith' AND first_name LIKE '%J' AND dob = '1976-12-23'
  这个查询只能使用索引的前两列，因为LIKE是一个范围条件。
  如果范围查询列值数量有限，那么可以通过使用多个等于条件来替换范围条件。

有些限制并不是B-Tree本身导致的，而是 MySQL 优化器和存储引擎使用索引的方式导致的，
这部分的限制有可能在未来的版本中会消除。
```
#### Hash index
```md
基于Hash表实现，只有精确匹配索引是由列的查询才有效。
在MySQL中，只有 Memory 引擎显式支持哈希索引，也是Memory的默认索引类型。
值得一提的是，它支持非唯一的哈希索引，这在数据库世界里比较与众不同。
如果哈希值相同，索引会以链表的方式存放多个记录指针到同一个哈希条目中。

Memory 也支持 B-Tree 索引，
```
#### 空间数据索引（R-Tree）
```md
MyISAM支持空间索引，可以作为地理数据存储。
和B-Tree索引不同，这类索引无须前缀查询，空间索引会从所有维度来索引数据。

查询时，可以有效地使用任意维度来组合查询。
必须使用MySQL的GIS相关函数如MBRECONTAINS()等来维护数据。
MySQL的GIS支持并不完善，所以大部分人不会使用该特性。

开源关系数据库中GIS解决方案做的比较好的是 PostgreSQL 的PostGIS。
```
#### 全文索引
```md
查找的是文本中的关键字，而不是直接比较索引中的值。
全文索引更类似于搜索引擎做的事情，而不是简单的WHERE条件匹配。

全文索引适用于 MATCH AGAINST操作。
```
### 索引的优点
```md
1. 减少了需要扫描的数据量
2. 帮助避免排序和临时表
3. 将随机IO变为顺序IO
```

* “三星系统”
```md
《Relational Database Index Design and the Optimizers》一书，
详细介绍了 如何计算索引的成本和作用，如果评估查询速度、如何分析索引维护的代价和带来的好处等。

其中评价一个索引是否适合一个查询给出了“三星系统（three-star 系统）”模型：
索引将相关的记录放到一起获得一星，
如果索引中的数据顺序和查找中的排列顺序一致获得二星，
如果索引中的列包含了查询需要的全部列则获得“三星”。
```

* 索引是最好的解决方案吗？
```md
对于小表，大部分情况下简单的全表扫描更高效。
对于中大型的表，索引会非常有效。
对于特大型表，建立和使用索引的代价将随之增长，
这种情况下需要一种技术直接区分出查询需要的一组数据，而不是一条一条记录地匹配。
如 可以使用分区技术。

如果表的数量特别多，可以建立一个元数据信息表，用力查询需要用到的某些特征。
例如执行安歇需要聚合多个应用分布在多个表的数据的查询，
则需要记录“哪个用户的信息存储在哪个表中”的元数据，这样查询的时候就可以直接忽略不包含指定用户信息的表。
对于大型系统，这是一个常用技巧。事实上，Infobright 就是使用类似的实现。
对于TB级别的数据，定位单条记录的意义不大，所以经常会用块级元数据技术来代替索引。
```