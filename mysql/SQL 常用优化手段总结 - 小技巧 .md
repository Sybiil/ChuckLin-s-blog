# SQL 常用优化手段总结 - 小技巧

---

中国有句古话叫做：欲速则不达。在一口气学完了分析 SQL 语句的一般步骤，与索引的正确运用方式后小憩片刻。搭配上红茶与白兰地轻松享用下面这些小技巧吧！

##优化 Order by 语句
众所周知，针对大量数据进行排序费时费力。了解 Mysql 的排序方式是优化数据库排序性能的充分条件。

###索引顺序扫描
```sql
show index from test_index_table
```
![image_1bhd1bbnb91v1hn1fsq1e2e589.png-63.9kB][1]

在 test_index_table 上我们创建了4个索引。Btree 索引的实现原理决定了索引的数据一定是顺序排列的。如果查询语句能够遵循索引数据本身的排列数据进行检索的话，则 mysql 不需要额外的排序工作，操作效率极高。查询类型显示为 Using index。

```sql
explain SELECT age  FROM test.test_index_table where age = '26' order by age 
```
![image_1bhd1paf01a7ed63slho0qcem.png-50.2kB][2]

###filesort 排序
若碰巧你无法使用索引覆盖扫描，mysql 会将取得的数据在内存中，或者硬盘上对数据进行分块后排序。由于多了一次排序工作，处理效率相比索引覆盖扫描自然就大打折扣了。
```sql
explain SELECT age  FROM test.test_index_table where age = '26' order by id 

```
![image_1bhd2pcv7qpi1qa72jd12831ukl13.png-53.4kB][3]
上面的查询虽然使用了索引作为查询条件，但是却选择了采用 id 作为排序规则进行排序。
根据 age 与根据 id 对应的索引进行排序的结果显然不同，所以 mysql 选择了 Using index 后再进行 Using where 回表扫描后对取出的数据进行排序，增加了数据库负担。

###复合索引的顺序扫描
如果你的索引为复合索引，这个问题将变的较为复杂一些。如下表中有 NAME_ADDESS 复合索引。
![image_1bhd338e6djrvd01idsvu51v151g.png-45.7kB][4] 数据按照 name、address 的顺序进行排序。

####复合索引索引覆盖查询
排序条件为 name 时，成功利用索引优化性能
```sql
explain SELECT name  FROM test.test_index_table where name = ' 张三' order by  name
```
![image_1bhd3s5nv178rm7i1ld3tk21cl11t.png-52.1kB][5]

####复合索引回表扫描查询
排序条件与索引不相同时，查询类型变为回表扫描。
```sql
explain SELECT name  FROM test.test_index_table where name = ' 张三' order by  address
```
![image_1bhd3vsu2k261o45ct919j317lv2a.png-54.8kB][6]
可见，尽量减少额外的排序，通过索引直接返回有序的数据，是优化 Mysql 排序的关键所在。


##优化 Group By 语句
即使你没有显示的指定 order by 语句，mysql 在默认情况下会对所有 Group By col1，col2的字段进行排序。
```sql
explain SELECT *  FROM test_index_table group by age 
```
![image_1bhfmijtb10ak1bepsq0dakedm.png-50.3kB][7]

如果想要避免排序结果产生的消耗，则可以指定 order by null 禁止排序
```sql
explain SELECT *  FROM test_index_table group by age order by null
```
![image_1bhfmk2jl18ubrpohovrip1kuf13.png-49kB][8]


##优化 OR 条件 
含有 or 条件的查询语句，如果想要利用索引，则 or 之前的每个条件列都必须要用到对应的索引。mysql 在处理含有 or 语句的查询时，会对 or 的各个字段分别查询后的结果进行 UNION。

##优化分页查询
类似"limit 1000,20"是相当常见的分页请求。mysql 在处理此类排序时会查询出1020条记录，然后舍弃掉前1000条数据，代价较高。所以在进行一般分页查询时候，在索引上完成排序分页的操作。尽量减少 mysql 排序时操作的数据量，是一种较为通用的优化思路。

```sql
explain SELECT * FROM test.test_index_table order by age limit 10000,20
```
![image_1bhkmer3lr4q1ila1tapg3d1g2l9.png-51kB][9]

在数据量巨大的表中使用 limit 分页是相当可怕的一件事情。针对这样的处理，我们就可以按照覆盖索引的方式尽量减少排序时 mysql 操作的数据量。

```sql
explain select * from test_index_table a inner join (SELECT id FROM test.test_index_table order by id limit 10000,20) b
on a.id = b.id
```
![image_1bhknak3k1ion1uas1iuk610lej13.png-69.2kB][10]


##总结
本问提及的优化手段虽然十分浅显，但是却至关重要。一个 web 工程大部分的性能问题都是由 database IO 引起的。对 SQL 语句进行适当的合理优化能够解决一个 web 工程80%的性能问题。所以乘着下午茶时间还没有结束，赶快再复习复习这些小技巧吧。

---
*参考书籍*
*《深入浅出 mysql》*




  [1]: http://static.zybuluo.com/mikumikulch/bf4ifvp3ybv0zx3w6wn6wdt3/image_1bhd1bbnb91v1hn1fsq1e2e589.png
  [2]: http://static.zybuluo.com/mikumikulch/lu0dbk9im9j3dzwcybrtveos/image_1bhd1paf01a7ed63slho0qcem.png
  [3]: http://static.zybuluo.com/mikumikulch/93spq2fq4dvpjrwi562vhal7/image_1bhd2pcv7qpi1qa72jd12831ukl13.png
  [4]: http://static.zybuluo.com/mikumikulch/x3k5lm48scquuuarg622i7y9/image_1bhd338e6djrvd01idsvu51v151g.png
  [5]: http://static.zybuluo.com/mikumikulch/goe5asm6o973zgs9qivli4pz/image_1bhd3s5nv178rm7i1ld3tk21cl11t.png
  [6]: http://static.zybuluo.com/mikumikulch/eb4y309eybycpzqckwzipu2c/image_1bhd3vsu2k261o45ct919j317lv2a.png
  [7]: http://static.zybuluo.com/mikumikulch/llkjyfplzuxj74divd2ejs52/image_1bhfmijtb10ak1bepsq0dakedm.png
  [8]: http://static.zybuluo.com/mikumikulch/lj8ue1r5q3nfmvcca8fvu8z8/image_1bhfmk2jl18ubrpohovrip1kuf13.png
  [9]: http://static.zybuluo.com/mikumikulch/76xzm5psgz9f31ygbjpfsv56/image_1bhkmer3lr4q1ila1tapg3d1g2l9.png
  [10]: http://static.zybuluo.com/mikumikulch/pkoun1q19wbxccetkfwhk8lb/image_1bhknak3k1ion1uas1iuk610lej13.png