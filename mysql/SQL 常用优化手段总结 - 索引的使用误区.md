# SQL 常用优化手段总结 - 索引的使用误区


---
回顾上一章[索引的应用](https://www.zybuluo.com/mikumikulch/note/750212)的内容，除了介绍了基本的使用索引优化 sql 语句的基本手法以外，还提到了滥用索引会引起性能恶化的问题。本章节的内容将会举例说明哪些场景的索引属于滥用，以及如何避免索引的误用。


##索引的滥用
为了优化 sql 语句的执行效率，我们往往会给表中加上各种各样的索引。
但是在某些场景下索引不仅无法被用来提升效率，还会增加表空间占用，延长 modify 方法的执行时间。
**下面是一些存在索引但无法使用的典型场景。**


###以%开头的 Like 查询

在上一章的内容中我们已经有所体会不遵循左前缀原则的查询统统无法使用 B-tree 索引。
那么针对 %Like 的查询我们有什么优化手段呢？来看下面这个例子：

表 test_index_table 拥有联合索引 NAME_ADDRESS 如下：

```sql
CREATE TABLE `test_index_table` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(45) DEFAULT NULL,
  `birthday` datetime DEFAULT NULL,
  `address` varchar(45) DEFAULT NULL,
  `phone` varchar(45) DEFAULT NULL,
  `note` varchar(45) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `NAME_ADDRESS` (`name`,`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=283 DEFAULT CHARSET=utf8
```
    有一种轻量级的解决方式为，利用索引比表小，扫描索引比扫描全表更快的原理，通过模糊匹配查询出符合条件的索引列表，之后再根据查询出的索引列表，回表进行连表检索。这样就能避免全表扫描时产生的大量 IO 请求。

```sql
explain select * from  (select id from test_index_table where name = '%四') as a, test_index_table as b
where a.id = b.id

```

![image_1bgofgp7r1ci2btk1q321ud81f4b1g.png-63.1kB][1]

    内层的子查询的 Using index 代表索引覆盖扫描。扫描出来的 id 与外表做关联，再查出相应的结果。
    
从结果分析可以看出两个 Sql 都利用了索引，所以理论上要比直接使用 name 来做全表扫描更快一些。

###数据类型出现隐式转换时也不会使用索引

让我们对上一个例子中的表增加一个 AGE 索引。
```sql
CREATE TABLE `test_index_table` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(45) DEFAULT NULL,
  `birthday` datetime DEFAULT NULL,
  `address` varchar(45) DEFAULT NULL,
  `phone` varchar(45) DEFAULT NULL,
  `note` varchar(45) DEFAULT NULL,
  `age` varchar(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `NAME_ADDRESS` (`name`,`id`) USING BTREE,
  KEY `AGE` (`age`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=283 DEFAULT CHARSET=utf8
```
尝试使用下面的 sql 语句进行查询
```sql
explain SELECT * FROM test.test_index_table where age = 26
```
由于表中的 age 是 VARCHAR 类型。而在 sql 语句中我们使用的是数字类型 26。MYSQL 默认会把输入的常量值进行转换以后才进行检索。现在我们通过 explain 看看这个语句的分析结果。
![image_1bgr0r2s9inlnb7o71g30v79.png-43.5kB][2]
很遗憾，索引并没有起到相应的作用。分析结果当前查询执行了全表扫描。

尝试将数字改为字符串重新执行 sql
```sql
explain SELECT * FROM test.test_index_table where age = 26
```
![image_1bgr0uotkjd34hhj3a1lkc1ksr9.png-29.9kB][3]
索引重新恢复了作用。可见，隐式的类型转换出现时，索引不会被使用。

    并非所有的隐式类型转换都无法使用索引。当列为 int、而查询语句中的 age 为字符串时，索引照样能够被使用。你可以试试对上面表中的索引 id 列进行查询，证明这个结论。
    
###使用 or 分割条件时，若 or 前的条件中的列有索引，而后面的列中没有索引，则前后涉及的索引都不会被用到。

```sql
explain SELECT * FROM test.test_index_table where id = '145' or address = '北京'
```
![image_1bgr1h59m1skq1e141054k911cim.png-44.3kB][4]
由于 address 没有索引，所以后面的查询一定会通过全表扫描执行。在存在全表扫描的情况下，就没必要多一次索引扫描，增加 IO 访问了。


##查看索引使用情况
掌握了索引的正确使用方法后是不是有点跃跃欲试的感觉了？别着急， mysql 还提供了一个针对数据库的索引有效性分析工具。在尝试优化索引之前最好通过此工具对数据库目前的索引效率进行一个全面的大致了解。
```sql
show status like 'handler_read%';
```
![image_1bhcscfvtas9db6a1s1g3p1l8f9.png-49.6kB][5]

Handler_read_key 的值代表了一个行被索引值读的次数。若值较高则意味着索引高效。若值较低则意味着增加索引所带来的性能改善不够理想。Handler_read_rnd_next 值的含义为：在数据文件中读取下一行的请求数。如果你的库进行了大量的全表扫描，则该值通常较高。这意味着查询运行的语句运行较为低效，应该马上建立索引进行补救。


##总结
上面举例了三个有代表性的索引被误用的场景。建议在平时的工作与学习中，可以将比较具有代表性的索引使用与误用的场景都记下来，整理成前后连贯的文章以加深理解和方便日后进行查询与再利用。

最后，索引在精不在多。对于某些经常被访问的查询我们可以尝试使用索引来优化执行效率。而对于某个速度很慢但是为系统管理员设计的查询，是否需要加上索引就需要结合实际情况判断了！


参考书籍：
- 《深入浅出 Mysql》

---

Copyright 2017/08/15 by Chuck Lin

若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！

![alipay.jpg-17.7kB][99]
![wechat.jpg-16.7kB][98]


[99]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[98]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg






    



  [1]: http://static.zybuluo.com/mikumikulch/7u8flbxq88o7dxnx9kgzji25/image_1bgofgp7r1ci2btk1q321ud81f4b1g.png
  [2]: http://static.zybuluo.com/mikumikulch/kvhx1z2hj5wt6a8e64zhvj0p/image_1bgr0r2s9inlnb7o71g30v79.png
  [3]: http://static.zybuluo.com/mikumikulch/hdf0iuezwpfpktiw127i8hff/image_1bgr0uotkjd34hhj3a1lkc1ksr9.png
  [4]: http://static.zybuluo.com/mikumikulch/3we8nh5pni28clxayqvujuuo/image_1bgr1h59m1skq1e141054k911cim.png
  [5]: http://static.zybuluo.com/mikumikulch/inaqs9cqp81ii5eqwcfdswfn/image_1bhcscfvtas9db6a1s1g3p1l8f9.png