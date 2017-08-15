# SQL 常用优化手段总结 - 分析 SQL 语句的一般步骤

---

数据库的性能调优是一个很大的话题。但是对于开发人员来讲，掌握一些常用的 SQL 优化手段却不是什么难事。
从本章节开始，将总结常用的适合于开发人员的 SQL 优化手段与大家分享。

要想解决性能优化的问题，首先要想办法发现哪些 SQL 有性能问题。通过下面这几个手段可以比较准确的定位到有问题的 SQL 进行分析优化。

##Show status 命令了解各种 SQL 的执行频率
在每一个 mysql session 中都可以使用 show status 命令查看当前数据库服务器的状态信息。如下所示：
![image_1bf763k1rs9j4kb66v1bnb1e1u9.png-75.2kB][1]

Com_xxx 表示每个 xxx 语句执行的次数。具体每个参数的意思你可以通过官方手册进行查询。总而言之，show status 向你展示了当前 mysql 服务器的运行状态。

###慢查询日志
慢查询日志是 mysql 自带的几种日志中的一种。它会自动的记录任何查询时间超出你设置的自定义时间的 sql 语句到你指定的日志目录中。通过慢查询日志可以定位到执行效率较低的具体的 SQL 语句。
打开慢查询日志的方法等由于每个版本的 mysql 都有不同，请查阅相关的官方文档。

##Explain
通过慢查询日志，我们可以查询到效率低下的 SQL 语句。对于这些 SQL语句又应该如何去分析它问题出在哪里呢？
![image_1bf77g112l9c1aqd1g431vuncu6m.png-40.4kB][2]
explain select 语句所返回的结果包含了该查询语句的执行细节。
具体每一个列的含义可对照官方手册。这里我们只提最主要的一个列的含义【type】：访问类型。
常见的访问类型如下：

- all 全表扫描
- index 索引全表扫描
- range 索引范围扫描 如： <、>、between 等操作符
- ref 使用非唯一索引扫描或前缀扫描。
- eq-ref 
- const、system 当检索条件对应到索引中为固定的值时，查询类型为 const。当查询的表只有一行时，查询类型为 system。
- NULL 不需要检索表就能得到结果，如 select 1 

从上到下，性能由最差到最好。一般来说，至少也要优化语句达到 range 这个等级。

##show profile 分析 SQL
show profle 能够在做 SQL 优化时帮助我们了解时间都耗费到哪里去了。
通常来说，对于一般的开发人员来讲 Explain 已经足够解决问题了。对于相关 show profile 的信息请查阅 mysql 官方手册。

##trace 分析优化器
mysql 5.6提供了对 SQL 的跟踪 trace。该功能进一步展示了优化器是如何选择执行计划的。对于开发人员来讲，explain 是最常用的分析手段。若对 trace 有兴趣请查阅相关的 mysql 官方手册。

##总结
性能优化最忌讳之一便是无用功优化。准确的找出问题 SQL 以及分析其原因，是优化 SQL 语句的第一步也是最重要的一步。


---


Copyright 2017/08/15 by Chuck Lin

若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！
![alipay.jpg-17.7kB][99]
![wechat.jpg-16.7kB][98]


[99]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[98]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg

  [1]: http://static.zybuluo.com/mikumikulch/r8kng6y5uh5bv0368tgww9g3/image_1bf763k1rs9j4kb66v1bnb1e1u9.png
  [2]: http://static.zybuluo.com/mikumikulch/ijvquijfj0905hhq7wwk43eu/image_1bf77g112l9c1aqd1g431vuncu6m.png