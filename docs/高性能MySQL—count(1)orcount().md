# 高性能MySQL——count(*) 和 count(1)和count(列名)区别

摘自: 

https://cloud.tencent.com/developer/article/1401567

https://mp.weixin.qq.com/s/MCFHNOQnTtJ6MGVjM3DP4A

执行效果上：

- count(*)包括了所有的列，相当于行数，在统计结果的时候，不会忽略列值为NULL
- count(1)包括了所有列，用1代表代码行，在统计结果的时候，不会忽略列值为NULL
- count(列名)只包括列名那一列，在统计结果的时候，会忽略列值为空（这里的空不是只空字符串或者0，而是表示null）的计数，即某个字段值为NULL时，不统计。

执行效率上：

- 列名为主键，count(1)会比count(列名)快
- 列名不为主键，count(1)会比count(列名)快
- 如果表多个列并且没有主键，则 count(1) 的执行效率优于 count(*)
- 如果有主键，则 select count（主键）的执行效率是最优的
- 如果表只有一个字段，则 select count(*) 最优。



>  count(列名)某个字段值为NULL时，不统计 

如果问一个程序员[MySQL](https://cloud.tencent.com/product/cdb?from=10680)中SELECT COUNT(1)和SELECT COUNT(*)有什么区别，会有很多人给出这样的答案“SELECT COUNT(*)”最终会转化成“SELECT COUNT(1)，而SELECT COUNT(1)省略了转换的这一步，所以SELECT COUNT(1)效率更高“，甚至有一些面试官也会给出类似的答案。最近在看一些历史遗留代码，绝大多数统计数量的SQL都在用SELECT COUNT(1)，觉得有必要搞清楚这个问题。

首先，以我们最常见的两种数据库表引擎MyISAM和Innodb来讲。

## MyISAM

MyISAM在统计表的总行数的时候会很快，但是有个大前提，**不能加有任何WHERE条件**。这是因为：MyISAM对于表的行数做了优化，具体做法是有一个变量存储了表的行数，如果查询条件没有WHERE条件则是查询表中一共有多少条数据，MyISAM可以做到迅速返回，所以也解释了如果加WHERE条件，则该优化就不起作用了。细心的同学会发现，innodb的表也有这么一个存储了表行数的变量，但是很遗憾这个值是一个估计值，没有什么实际意义。

## Innodb

在该引擎下，COUNT(1)和COUNT(*)哪个快呢？结论是：这俩在高版本的MySQL（5.5及以后，5.1的没有考证）是没有什么区别的，也就没有COUN(1)会比COUNT(\*)更快这一说了。

WHY？这就要从COUNT()函数的具体含义说起了。”

>    COUNT()有两个非常不同的作用：它可以统计某个列值的数量，也可以统计行数。在统计列值时要求列值是非空的（不统计NULL）。如果在COUNT()的括号中定了列或者列表达式，则统计的就是这个表达式有值的结果数。......COUNT()的另一个作用是统计结果集的行数。当MySQL确认括号内的表达式值不可能为空时，实际上就是在统计行数。最简单的就是当我们使用COUNT(*)的时候，这种情况下通配符*并不像我们猜想的那样扩展成所有的列，实际上，他会忽略所有列而直接统计所有的行数“——《高性能MySQL》。  

*通常，我们将第一个字段（一般是ID）作为主键，那么这个时候COUNT(id)实际统计的就是行数，因为主键肯定是非NULL的。问题是Innodb是通过主键索引来统计行数的吗？结论是：如果该表只有一个主键索引，没有任何二级索引的情况下，那么COUNT(\*)和COUNT(1)都是通过通过主键索引来统计行数的。如果该表有二级索引，则COUNT(1)和COUNT(\*)都会通过**占用空间最小的字段的二级索引**进行统计，也就是说虽然*COUNT(id)指定了主键列,但是innodb不会真的去统计主键索引。

## 实验

## 第一步

新建一张基于Innodb的表，只有一个ID主键，并插入5w的测试数据，建表语句如下：

```mysql
CREATE TABLE `tb_news` (
  `id` bigint(21) NOT NULL AUTO_INCREMENT,
  `title` varchar(50) NOT NULL,
  `content` mediumtext NOT NULL,
  `count_ass` char(1) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=50001 DEFAULT CHARSET=utf8
```

这个时候执行COUNT(1)和COUNT(*)可以看到解释器的结果如下（两者一致，所以就只截了一张图），可以看到，两者都用了主键索引进行行数的统计：

![eub848elm8](https://gitee.com/HNov/image/raw/master/typora/20200526113809.jpeg)

## 第二步

新建一个二级索引title，之后在分别看一下COUNT(1)和COUNT(*)的解释器结果（两者依然完全一致），这时已经用二级索引进行统计而非主键索引：

![3wid587yqs](https://gitee.com/HNov/image/raw/master/typora/20200526114108.jpeg)

## 第三步

在我们之前特地预留的一个小字段count_ass字段建一个索引，到这一步目前表中有三个索引：一个主键索引，两个二级索引。

![g1fepa2xl0](https://gitee.com/HNov/image/raw/master/typora/20200526114138.jpeg)

## 原理

目前基于磁盘的数据库或者搜索引擎（比如Lucene）的性能瓶颈主要都是在IO阶段，相比于CPU和RAM，IO操作实在太慢了，所以这类系统的优化方向也都都是类似的——尽一切可能减少IO的次数（所以很多用ES的程序在性能优化到极限的时候选择直接上SSD）。这里统计行数的操作，查询优化器的优化方向就是选择能够让IO次数最少的索引，也就是基于占用空间最小的字段所建的索引（每次IO读取的数据量是固定的，索引占用的空间越小所需的IO次数也就越少）。而Innodb的主键索引是聚簇索引（包含了KEY，除了KEY之外的其他字段值，事务ID和MVCC回滚指针）所以主键索引一定会比二级索引（包含KEY和对应的主键ID）大，也就是说在有二级索引的情况下，一般COUNT()都不会通过主键索引来统计行数，在有多个二级索引的情况下选择占用空间最小的。

如果说有张Innodb的表只有主键索引，而且记录还比较大（比如30K），则统计行的操作会非常慢，因为IO次数会很多（这里就不做实验截图了，有兴趣可以自己试一下）。

一个优化方案就是预先建一个小字段并建二级索引专门用来统计行数，极端情况下这种优化速度提高上千倍也是正常的。

## 结论

结论就是对于COUNT(1)和COUNT(\*)执行优化器的优化是完全一样的，并没有COUNT(1)会比COUNT(*)快这个说法。

即count(字段)<count(主键 id)<count(1)≈count(*)