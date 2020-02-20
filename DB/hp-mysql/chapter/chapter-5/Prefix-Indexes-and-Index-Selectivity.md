## 前缀索引和索引选择性
* 索引选择性
```md
不重复的索引值（也称为基数，cardinality）和数据表的记录总数（#T）的比值。
索引的选择性越高，查询的效率越高，唯一索引的选择性是1，性能时最好的。
```
* 前缀索引
```md
有时候需要索引很长的字符列，这会让索引变得慢且大，一个策略是模拟哈希索引。
还可以使用前缀索引，只索引开始的部分字符，可以节约索引空间，从而提高索引效率。
但也会降低索引的选择性。

一般情况下，某个列的前缀的选择性也是足够高的，足以满足查询性能。
对于 BLOB、TEXT 或者很长的VARCHAR类型的列，必须使用前缀索引，因为MySQL不允许索引这些列的完整长度。

重点在于选择足够长的前缀以保证较高的选择性。
目标是使得前缀索引的选择性接近于索引整个列，即前缀的“基数”接近于完整列的“基数”。
```
* 实践
```md
其思想是增加前缀长度，直到前缀变得几乎与列的全长一样具有选择性。

一般找到最常见的列表，然后和最常见的前缀列表进行比较。
如：
首先找最常见的城市列表：
mysql> SELECT COUNT(*) AS cnt, city
-> FROM sakila.city_demo GROUP BY city ORDER BY cnt DESC LIMIT 10;
     +-----+----------------+
     | cnt | city           |
     +-----+----------------+
     |  65 | London         |
     |  49 | Hiroshima      |
     |  48 | Teboksary      |
     |  48 | Pak Kret       |
     |  48 | Yaound         |
     |  47 | Tel Aviv-Jaffa |
     |  47 | Shimoga        |
     |  45 | Cabuyao        |
     |  45 | Callao         |
     |  45 | Bislig         |
     +-----+----------------+

查找最频繁出现的城市前缀，先从3个前缀字符开始：
mysql> SELECT COUNT(*) AS cnt, LEFT(city, 3) AS pref
-> FROM sakila.city_demo GROUP BY pref ORDER BY cnt DESC LIMIT 10;
     +-----+------+
     | cnt | pref |
     +-----+------+
     | 483 | San  |
     | 195 | Cha  |
     | 177 | Tan  |
     | 167 | Sou  |
     | 163 | al-  |
     | 163 | Sal  |
     | 146 | Shi  |
     | 136 | Hal  |
     | 130 | Val  |
     | 129 | Bat  |
     +-----+------+
继续增加前缀长度，经过试验发现前缀长度 7 比较合适：
mysql> SELECT COUNT(*) AS cnt, LEFT(city, 7) AS pref
-> FROM sakila.city_demo GROUP BY pref ORDER BY cnt DESC LIMIT 10;
    +-----+---------+
    | cnt | pref    |
    +-----+---------+
    |  70 | Santiag |
    |  68 | San Fel |
    |  65 | London  |
    |  61 | Valle d |
    |  49 | Hiroshi |
    |  48 | Teboksa |
    |  48 | Pak Kre |
    |  48 | Yaound  |
    |  47 | Tel Avi |
    |  47 | Shimoga |
    +-----+---------+
注解：上表的输出和原始值统计是相近的。

另外一种计算方法是计算整列的选择性，并使前缀的选择性接近于完整列的选择性。
计算完整列的选择性：
mysql> SELECT COUNT(DISTINCT city)/COUNT(*) FROM sakila.city_demo; 
+-------------------------------+
| COUNT(DISTINCT city)/COUNT(*) | 
+-------------------------------+
|                        0.0312 |
+-------------------------------+
通常来说（也有例外），如果前缀的选择性能够接近 0.031，基本上就可用了。
可以再一个查询中针对不同前缀长度进行计算，这对于大表非常有用。
计算不同前缀长度的选择性：
mysql> SELECT COUNT(DISTINCT LEFT(city, 3))/COUNT(*) AS sel3,
-> COUNT(DISTINCT LEFT(city, 4))/COUNT(*) AS sel4,
-> COUNT(DISTINCT LEFT(city, 5))/COUNT(*) AS sel5,
-> COUNT(DISTINCT LEFT(city, 6))/COUNT(*) AS sel6,
-> COUNT(DISTINCT LEFT(city, 7))/COUNT(*) AS sel7
-> FROM sakila.city_demo; 
+--------+--------+--------+--------+--------+ 
| sel3   | sel4   | sel5   | sel6   | sel7   | 
+--------+--------+--------+--------+--------+ 
| 0.0239 | 0.0293 | 0.0305 | 0.0309 | 0.0310 | 
+--------+--------+--------+--------+--------+
```
* 最坏情况下的选择性
```md
平局选择性会让你认为 sel4 和 sel5的索引已经足够了，但是数据如果分布不均，可能会有陷阱。
如果观察前缀为 4 的最长出现城市的次数，可以看到分布明显不均：
mysql> SELECT COUNT(*) AS cnt, LEFT(city, 4) AS pref
-> FROM sakila.city_demo GROUP BY pref ORDER BY cnt DESC LIMIT 5;
     +-----+------+
     | cnt | pref |
     +-----+------+
     | 205 | San  |
     | 200 | Sant |
     | 135 | Sout |
     | 104 | Chan |
     |  91 | Toul |
     +-----+------+

对于以“San” 开头的城市的选择性就会非常糟糕，因为很多城市都以这个词开头。
```
* 创建前缀索引
```md
mysql> ALTER TABLE sakila.city_demo ADD KEY (city(7));
```
* 前缀索引总结
```md
前缀索引是一种能使索引更小、更快的有效办法。

常见场景：
针对很长的十六进制唯一ID使用前缀索引。
此时如果采用长度为 8 的前缀索引，通常能显著提升性能，并且对上层应用完全透明。

缺点：
MySQL无法使用前缀索引做ORDER BY 和 GROUP BY，也无做覆盖扫描。
```
* 后缀索引
```md
一些场景下，后缀索引也很有用途，如 查找某个域名的所有电子邮件地址。
```
> <font color=DarkSalmon size=3> MySQL 原生不支持反向索引，但是可以把这个字符串反转后存储，并创建前缀索引来解决。可以通过触发器来维护这种索引，参考 5.1节中“创建自定义Hash索引”</font>
