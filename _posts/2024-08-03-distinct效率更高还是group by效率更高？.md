---
layout:       post
title:        "distinct效率更高还是group by效率更高？"
author:       "Joey"
header-style: text
catalog:      true
tags:
    - MySQL
---

> 来源CSDN

先说大致的结论（完整结论在文末）：

> 在语义相同，有索引的情况下：group by和distinct都能使用索引，效率相同。
> 
> 在语义相同，无索引的情况下：distinct效率高于group by。原因是distinct 和 group by都会进行分组操作，但group by可能会进行排序，触发filesort，导致sql执行效率低下。

### distinct的使用

#### distinct用法

```sql
SELECT DISTINCT columns FROM table_name WHERE where_conditions;
```

例如：

```sql
mysql> select distinct age from student;
+------+
| age  |
+------+
|   10 |
|   12 |
|   11 |
| NULL |
+------+
4 rows in set (0.01 sec)
```


DISTINCT 关键词用于返回唯一不同的值。放在查询语句中的第一个字段前使用，且**作用于主句所有列**。

如果列具有NULL值，并且对该列使用DISTINCT子句，MySQL将保留一个NULL值，并删除其它的NULL值，因为DISTINCT子句将所有NULL值视为相同的值。

#### distinct多列去重

 distinct多列的去重，则是根据指定的去重的列信息来进行，即只有**所有指定的列信息都相同**，才会被认为是重复的信息。

```sql
SELECT DISTINCT column1,column2 FROM table_name WHERE where_conditions;

mysql> select distinct sex,age from student;
+--------+------+
| sex    | age  |
+--------+------+
| male   |   10 |
| female |   12 |
| male   |   11 |
| male   | NULL |
| female |   11 |
+--------+------+
5 rows in set (0.02 sec)
```

### group by的使用

 对于基础去重来说，group by的使用和distinct类似:

#### 单列去重

语法：

```sql
SELECT columns FROM table_name WHERE where_conditions GROUP BY columns;
```

执行：

```sql
mysql> select age from student group by age;
+------+
| age  |
+------+
|   10 |
|   12 |
|   11 |
| NULL |
+------+
4 rows in set (0.02 sec)
```

#### 多列去重

语法：

```sql
SELECT columns FROM table_name WHERE where_conditions GROUP BY columns;
```

执行：

```sql
mysql> select sex,age from student group by sex,age;
+--------+------+
| sex    | age  |
+--------+------+
| male   |   10 |
| female |   12 |
| male   |   11 |
| male   | NULL |
| female |   11 |
+--------+------+
5 rows in set (0.03 sec)
```

#### 区别示例

 两者的语法区别在于，group by可以进行单列去重，group by的原理是先对结果进行分组排序，然后返回**每组中的第一条**数据。且是根据group by的后接字段进行去重的。
例如：

```sql
mysql> select sex,age from student group by sex;
+--------+-----+
| sex    | age |
+--------+-----+
| male   |  10 |
| female |  12 |
+--------+-----+
2 rows in set (0.03 sec)
```

### distinct和group by原理

在大多数例子中，DISTINCT可以被看作是特殊的GROUP BY，它们的实现都基于分组操作，且都可以通过**松散索引扫描、紧凑索引扫描**来实现。

DISTINCT和GROUP BY都是**可以使用索引进行扫描搜索**的。例如以下两条sql（只单单看表格最后extra的内容），我们对这两条sql进行分析，可以看到，在extra中，这两条sql都使用了紧凑索引扫描`Using index for group-by`。所以，在一般情况下，对于相同语义的DISTINCT和GROUP BY语句，我们可以对其使用相同的索引优化手段来进行优化。

```sql
mysql> explain select int1_index from test_distinct_groupby group by int1_index;
+----+-------------+-----------------------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
| id | select_type | table                 | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-----------------------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | test_distinct_groupby | NULL       | range | index_1       | index_1 | 5       | NULL |  955 |   100.00 | Using index for group-by |
+----+-------------+-----------------------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
1 row in set (0.05 sec)

mysql> explain select distinct int1_index from test_distinct_groupby;
+----+-------------+-----------------------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
| id | select_type | table                 | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-----------------------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | test_distinct_groupby | NULL       | range | index_1       | index_1 | 5       | NULL |  955 |   100.00 | Using index for group-by |
+----+-------------+-----------------------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
1 row in set (0.05 sec)

```

 但对于GROUP BY来说，在MYSQL8.0之前，GROUP Y默认会依据字段进行**隐式排序**。
 可以看到，下面这条sql语句在使用了临时表的同时，还进行了filesort。

```sql
mysql> explain select int6_bigger_random from test_distinct_groupby GROUP BY int6_bigger_random;
+----+-------------+-----------------------+------------+------+---------------+------+---------+------+-------+----------+---------------------------------+
| id | select_type | table                 | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                           |
+----+-------------+-----------------------+------------+------+---------------+------+---------+------+-------+----------+---------------------------------+
|  1 | SIMPLE      | test_distinct_groupby | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 97402 |   100.00 | Using temporary; Using filesort |
+----+-------------+-----------------------+------------+------+---------------+------+---------+------+-------+----------+---------------------------------+
1 row in set (0.04 sec)
```

#### 隐式排序

 对于隐式排序，我们可以参考Mysql官方的解释：

[MySQL :: MySQL 5.7 Reference Manual :: 8.2.1.14 ORDER BY Optimization](https://dev.mysql.com/doc/refman/5.7/en/order-by-optimization.html)

>  GROUP BY 默认隐式排序（指在 GROUP BY 列没有 ASC 或 DESC 指示符的情况下也会进行排序）。然而，GROUP BY进行显式或隐式排序已经过时（deprecated）了，要生成给定的排序顺序，请提供 ORDER BY 子句。

所以，在Mysql8.0之前,Group by会**默认**根据作用字段（Group by的后接字段）对结果进行**排序**。在能利用索引的情况下，Group by不需要额外进行排序操作；但当无法利用索引排序时，Mysql优化器就不得不选择通过使用临时表然后再排序的方式来实现GROUP BY了。且当结果集的大小超出系统设置临时表大小时，Mysql会将临时表数据copy到磁盘上面再进行操作，语句的执行效率会变得极低。这也是Mysql选择将此操作（隐式排序）**弃用**的原因。


基于上述原因，Mysql在8.0时，对此进行了优化更新：

[MySQL :: MySQL 8.0 Reference Manual :: 8.2.1.16 ORDER BY Optimization](https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html)

> 从前（Mysql5.7版本之前），Group by会根据确定的条件进行隐式排序。在mysql 8.0中，已经移除了这个功能，所以不再需要通过添加order by null 来禁止隐式排序了，但是，查询结果可能与以前的 MySQL 版本不同。要生成给定顺序的结果，请按通过ORDER BY指定需要进行排序的字段。

因此，我们的结论也出来了：

> 在语义相同，有索引的情况下：group by和distinct都能使用索引，效率相同。因为group by和distinct近乎等价，distinct可以被看做是特殊的group by。
> 
> 在语义相同，无索引的情况下：distinct效率高于group by。原因是distinct 和 group by都会进行分组操作，但group by在Mysql8.0之前会进行隐式排序，导致触发filesort，sql执行效率低下。但从Mysql8.0开始，Mysql就删除了隐式排序，所以，此时在语义相同，无索引的情况下，group by和distinct的执行效率也是近乎等价的。

### 推荐group by的原因

1.group by语义更为清晰
2.group by可对数据进行更为复杂的一些处理

相比于distinct来说，group by的语义明确。且由于distinct关键字会对所有字段生效，在进行复合业务处理时，group by的使用灵活性更高，group by能根据分组情况，对数据进行更为复杂的处理，例如通过having对数据进行过滤，或通过聚合函数对数据进行运算。
