本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

# 1.简介

​     使用EXPLAIN关键字可以模拟优化器执行SQL查询语句，从而知道MySQL是如何处理你的SQL语句的。分析你的查询语句或是表结构的性能瓶颈。在日常工作中，我们会有时会开慢查询去记录一些执行时间比较久的SQL语句，找出这些SQL语句并不意味着完事了，有时我们常常用到explain这个命令来查看一个这些SQL语句的执行计划。

​     官网地址：https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

​     测试版本：8.0.26

##   1.1 explain的主要作用

​     1、表的读取顺序

​     2、数据读取操作的操作类型

​     3、哪些索引可以使用

​     4、哪些索引被实际使用

​     5、表之间的引用

​     6、每张表有多少行被优化器查询

##    1.2 SQL执行顺序

​     下面是一条完成的sql：

```mysql
select distinct 
        <select_list>
from
    <left_table><join_type>
join <right_table> on <join_condition>
where
    <where_condition>
group by
    <group_by_list>
having
    <having_condition>
order by
    <order_by_condition>
limit <limit number>
```

执行顺序：

```
1、from <left_table><join_type>
2、on <join_condition>
3、<join_type> join <right_table>
4、where <where_condition>
5、group by <group_by_list>
6、having <having_condition>
7、select
8、distinct <select_list>
9、order by <order_by_condition>
10、limit <limit_number>
```

Sql执行步骤：

第一步：首先对from子句中的前两个表执行一个笛卡尔乘积，此时生成虚拟表 vt1（选择相对小的表做基础表）。 

﻿

第二步：接下来便是应用on筛选器，on 中的逻辑表达式将应用到 vt1 中的各个行，筛选出满足on逻辑表达式的行，生成虚拟表 vt2 。

﻿

第三步：如果是outer join 那么这一步就将添加外部行，left outer jion 就把左表在第二步中过滤的添加进来，如果是right outer join 那么就将右表在第二步中过滤掉的行添加进来，这样生成虚拟表 vt3 。

﻿

第四步：如果 from 子句中的表数目多于两个表，那么就将vt3和第三个表连接从而计算笛卡尔乘积，生成虚拟表，该过程就是一个重复1-3的步骤，最终得到一个新的虚拟表 vt3。 

﻿

第五步：应用where筛选器，对上一步生产的虚拟表引用where筛选器，生成虚拟表vt4，在这有个比较重要的细节不得不说一下，对于包含outer join子句的查询，就有一个让人感到困惑的问题，到底在on筛选器还是用where筛选器指定逻辑表达式呢？on和where的最大区别在于，如果在on应用逻辑表达式那么在第三步outer join中还可以把移除的行再次添加回来，而where的移除的最终的。举个简单的例子，有一个学生表（班级,姓名）和一个成绩表(姓名,成绩)，我现在需要返回一个x班级的全体同学的成绩，但是这个班级有几个学生缺考，也就是说在成绩表中没有记录。为了得到我们预期的结果我们就需要在on子句指定学生和成绩表的关系（学生.姓名=成绩.姓名）那么我们是否发现在执行第二步的时候，对于没有参加考试的学生记录就不会出现在vt2中，因为他们被on的逻辑表达式过滤掉了,但是我们用left outer join就可以把左表（学生）中没有参加考试的学生找回来，因为我们想返回的是x班级的所有学生，如果在on中应用学生.班级='x'的话，left outer join会把x班级的所有学生记录找回，所以只能在where筛选器中应用学生.班级='x' 因为它的过滤是最终的。 

﻿

第六步：group by 子句将中的唯一的值组合成为一组，得到虚拟表vt5。如果应用了group by，那么后面的所有步骤都只能得到的vt5的列或者是聚合函数（count、sum、avg等）。原因在于最终的结果集中只为每个组包含一行。这一点请牢记。 

﻿

第七步：应用cube或者rollup选项，为vt5生成超组，生成vt6. 

﻿

第八步：应用having筛选器，生成vt7。having筛选器是第一个也是为唯一一个应用到已分组数据的筛选器。 

﻿

第九步：处理select子句。将vt7中的在select中出现的列筛选出来。生成vt8. 

﻿

第十步：应用distinct子句，vt8中移除相同的行，生成vt9。事实上如果应用了group by子句那么distinct是多余的，原因同样在于，分组的时候是将列中唯一的值分成一组，同时只为每一组返回一行记录，那么所以的记录都将是不相同的。 

﻿

第十一步：应用order by子句。按照order_by_condition排序vt9，此时返回的一个游标，而不是虚拟表。sql是基于集合的理论的，集合不会预先对他的行排序，它只是成员的逻辑集合，成员的顺序是无关紧要的。对表进行排序的查询可以返回一个对象，这个对象包含特定的物理顺序的逻辑组织。这个对象就叫游标。正因为返回值是游标，那么使用order by 子句查询不能应用于表表达式。排序是很需要成本的，除非你必须要排序，否则最好不要指定order by，最后，在这一步中是第一个也是唯一一个可以使用select列表中别名的步骤。 

﻿

第十二步：应用top选项。此时才返回结果给请求者即用户。 

# 2.使用

在select语句前加上explain便可以进行分析，比如有如下sql语句：

```
explain select * from emp where name = 'tengyi';
```

执行结果：

结果中各个字段是什么意思呢？这里先贴出一个表格进行总述：

| 信息          | 描述                                                         |
| :------------ | :----------------------------------------------------------- |
| id            | 查询的序号，包含一组数字，表示查询中执行select子句或操作表的顺序**两种情况**id相同，执行顺序从上往下id不同，id值越大，优先级越高，越先执行 |
| select_type   | 查询类型，主要用于区别普通查询，联合查询，子查询等的复杂查询1、simple ——简单的select查询，查询中不包含子查询或者UNION2、primary ——查询中若包含任何复杂的子部分，最外层查询被标记3、subquery——在select或where列表中包含了子查询4、derived——在from列表中包含的子查询被标记为derived（衍生），MySQL会递归执行这些子查询，把结果放到临时表中5、union——如果第二个select出现在UNION之后，则被标记为UNION，如果union包含在from子句的子查询中，外层select被标记为derived6、union result:UNION 的结果 |
| table         | 输出的行所引用的表                                           |
| type          | 显示联结类型，显示查询使用了何种类型，按照从最佳到最坏类型排序<br />1、system：表中仅有一行（=系统表）这是const联结类型的一个特例。<br />2、const：表示通过索引一次就找到，const用于比较primary key或者unique索引。因为只匹配一行数据，所以如果将主键置于where列表中，mysql能将该查询转换为一个常量3、eq_ref:唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于唯一索引或者主键扫描<br />4、ref:非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它返回所有匹配某个单独值的行，可能会找多个符合条件的行，属于查找和扫描的混合体<br />5、range:只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是where语句中出现了between,in等范围的查询。这种范围扫描索引扫描比全表扫描要好，因为它开始于索引的某一个点，而结束另一个点，不用全表扫描<br />6、index:index 与all区别为index类型只遍历索引树。通常比all快，因为索引文件比数据文件小很多。<br />7、all：遍历全表以找到匹配的行注意:一般保证查询至少达到range级别，最好能达到ref。 |
| possible_keys | 指出MySQL能使用哪个索引在该表中找到行                        |
| key           | 显示MySQL实际决定使用的键(索引)。如果没有选择索引,键是NULL。查询中如果使用覆盖索引，则该索引和查询的select字段重叠。 |
| key_len       | 表示索引中使用的字节数，该列计算查询中使用的索引的长度在不损失精度的情况下，长度越短越好。如果键是NULL,则长度为NULL。该字段显示为索引字段的最大可能长度，并非实际使用长度。 |
| ref           | 显示索引的哪一列被使用了，如果有可能是一个常数，哪些列或常量被用于查询索引列上的值 |
| rows          | 根据表统计信息以及索引选用情况，大致估算出找到所需的记录所需要读取的行数 |
| Extra         | 包含不适合在其他列中显示，但是十分重要的额外信息<br />1、Using filesort：说明mysql会对数据适用一个外部的索引排序。而不是按照表内的索引顺序进行读取。MySQL中无法利用索引完成排序操作称为“文件排序”<br />2、Using temporary:使用了临时表保存中间结果，mysql在查询结果排序时使用临时表。常见于排序order by和分组查询group by。<br />3、Using index:表示相应的select操作用使用覆盖索引，避免访问了表的数据行。如果同时出现using where，表名索引被用来执行索引键值的查找；如果没有同时出现using where，表名索引用来读取数据而非执行查询动作。<br />4、Using where :表明使用where过滤<br />5、using join buffer:使用了连接缓存<br />6、impossible where:where子句的值总是false，不能用来获取任何元组<br />7、select tables optimized away：在没有group by子句的情况下，基于索引优化Min、max操作或者对于MyISAM存储引擎优化count（*），不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化。<br />8、distinct：优化distinct操作，在找到第一匹配的元组后即停止找同样值的动作。 |
| filtered      | 显示了通过条件过滤出的行数的百分比估计值                     |

﻿

文中用到的建表语句：

```
create table course(
cid int(3),
cname varchar(20),
tid int(3)
);
create table teacher(
tid int(3),
tname varchar(20),
tcid int(3)
);
create table teacherCard(
tcid int(3),
tcdesc varchar(200)
);
insert into course values(1,'java',1);
insert into course values(2,'html',1);
insert into course values(3,'sql',2);
insert into course values(4,'web',3);

insert into teacher values(1,'tz',1);
insert into teacher values(2,'tw',2);
insert into teacher values(3,'tl',3);

insert into teacherCard values(1,'tezdesc');
insert into teacherCard values(2,'twdesc');
insert into teacherCard values(3,'tldesc');
```

﻿

查询课程编号为2或教师证编号为3的老师信息：

```
select t.* from teacher t,course c,teacherCard tc where t.tid = c.tid and t.tcid = tc.tcid and (c.cid = 2 or tc.tcid = 3);

explain select t.* from teacher t,course c,teacherCard tc where t.tid = c.tid and t.tcid = tc.tcid and (c.cid = 2 or tc.tcid = 3);
```

﻿

```
mysql> explain select t.* from teacher t,course c,teacherCard tc where t.tid = c.tid and t.tcid = tc.tcid and (c.cid = 2 or tc.tcid = 3);
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL                                       |
|  1 | SIMPLE      | tc    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where; Using join buffer (hash join) |
|  1 | SIMPLE      | c     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |    25.00 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
3 rows in set, 1 warning (0.00 sec)
```

﻿

## 2.1 id

 **（1） id值相同，从上往下顺序执行。**--t3-tc3-c4

```
insert into teacher values(4,'ta',4);
insert into teacher values(5,'tb',5);
insert into teacher values(6,'tc',6);
explain select t.* from teacher t,course c,teacherCard tc where t.tid = c.tid and t.tcid = tc.tcid and (c.cid = 2 or tc.tcid = 3);
//tc3-t6-c4
```

```
mysql> insert into teacher values(4,'ta',4);
Query OK, 1 row affected (0.00 sec)

mysql> insert into teacher values(5,'tb',5);
Query OK, 1 row affected (0.00 sec)

mysql> insert into teacher values(6,'tc',6);
Query OK, 1 row affected (0.01 sec)

mysql> explain select t.* from teacher t,course c,teacherCard tc where t.tid = c.tid and t.tcid = tc.tcid and (c.cid = 2 or tc.tcid = 3);
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | tc    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL                                       |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |    16.67 | Using where; Using join buffer (hash join) |
|  1 | SIMPLE      | c     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |    25.00 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
3 rows in set, 1 warning (0.00 sec)
```



﻿

表的执行顺序会根据数据量的改变而改变。原因是：比较笛卡尔积的大小。—小的先执行。

--笛卡尔积的元素是元组，关系A和B的笛卡尔积可以记为(AXB)，如果A为a目，B为b目，那么A和B的笛卡尔积为(a+b)列、(a*b)行的元组集合。

--简单解释一下笛卡尔积：

比如有两个集合A和B。

A = {0,1}     B = {2,3,4}

集合 A×B 和 B×A的结果集就可以分别表示为以下这种形式：

A×B = {（0，2），（1，2），（0，3），（1，3），（0，4），（1，4）}；

B×A = {（2，0），（2，1），（3，0），（3，1），（4，0），（4，1）}；

以上A×B和B×A的结果就可以叫做两个集合相乘的‘笛卡尔积’。

从以上的数据分析我们可以得出以下两点结论：

1，两个集合相乘，不满足交换率，既 A×B ≠ B×A;

2，A集合和B集合相乘，包含了集合A中元素和集合B中元素相结合的所有的可能性。既两个集合相乘得到的新集合的元素个数是 A集合的元素个数 × B集合的元素个数;

﻿

验证：

```
delete from course where cid>2;
explain select t.* from teacher t,course c,teacherCard tc where t.tid = c.tid and t.tcid = tc.tcid and (c.cid = 2 or tc.tcid = 3);
```

```
mysql> delete from course where cid>2;
Query OK, 2 rows affected (0.00 sec)

mysql> explain select t.* from teacher t,course c,teacherCard tc where t.tid = c.tid and t.tcid = tc.tcid and (c.cid = 2 or tc.tcid = 3);
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | c     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |   100.00 | NULL                                       |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |    16.67 | Using where; Using join buffer (hash join) |
|  1 | SIMPLE      | tc    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
3 rows in set, 1 warning (0.01 sec)
```

**（2） id值不同，id值越大月优先查询（本质：在嵌套子查询时，先查内层，再差外层）**

查询教授SQL课程的老师的描述（desc）

```
explain select tc.tcdesc from teacherCard tc,course c,teacher t where c.tid = t.tid and t.tcid = tc.tcid and c.cname = 'sql';
//c-t-tc
```

将以上多表查询改成子查询形式：

```
explain select tc.tcdesc from teacherCard tc where tc.tcid = (select t.tcid from teacher t where t.tid = (select c.tid from course c where cname = 'sql'));
```

```
mysql> explain select tc.tcdesc from teacherCard tc,course c,teacher t where c.tid = t.tid and t.tcid = tc.tcid and c.cname = 'sql';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | c     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where                                |
|  1 | SIMPLE      | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |    16.67 | Using where; Using join buffer (hash join) |
|  1 | SIMPLE      | tc    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
3 rows in set, 1 warning (0.00 sec)

mysql>
mysql> explain select tc.tcdesc from teacherCard tc where tc.tcid = (select t.tcid from teacher t where t.tid = (select c.tid from course c where cname = 'sql'));
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | tc    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where |
|  2 | SUBQUERY    | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |    16.67 | Using where |
|  3 | SUBQUERY    | c     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
3 rows in set, 1 warning (0.00 sec)
```



**（3）id值有相同，有不同（子查询+多表形式），id值越大越优先；id值相同，从上往下顺序执行**

```
explain select t.tname,tc.tcdesc from teacher t,teacherCard tc where t.tcid = tc.tcid and t.tid = (select c.tid from course c where cname = 'sql');
//c-t-tc
```

```
mysql> explain select t.tname,tc.tcdesc from teacher t,teacherCard tc where t.tcid = tc.tcid and t.tid = (select c.tid from course c where cname = 'sql');
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
|  1 | PRIMARY     | t     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |    16.67 | Using where                                |
|  1 | PRIMARY     | tc    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where; Using join buffer (hash join) |
|  2 | SUBQUERY    | c     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where                                |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------------------+
3 rows in set, 1 warning (0.00 sec)
```



## 2.2 select_type:查询类型

PRIMARY :包含子查询SQL中的 主查询（最外层）

SUBQUERY:包含子查询SQL中的 子查询（非最外层）

SIMPLE:简单查询（不包含子查询、union）

DERIVED:衍生查询（使用到了临时表）

a.在from子查询中只有一张表

```
explain select cr.cname from (select * from course where tid in(1,2)) cr;
```

b.在from子查询中，如果有table1 union table2，则（左表）table1就是derived，（右表）table2就是union

```
 explain select cr.cname from (select * from course where tid = 1 union select * from course where tid = 2) cr;
```

```
mysql> explain select cr.cname from (select * from course where tid in(1,2)) cr;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | course | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql>  explain select cr.cname from (select * from course where tid = 1 union select * from course where tid = 2) cr;
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | PRIMARY      | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | NULL            |
|  2 | DERIVED      | course     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where     |
|  3 | UNION        | course     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where     |
| NULL | UNION RESULT | <union2,3> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
4 rows in set, 1 warning (0.01 sec)
```

﻿



**UNION**:上例中的table2就是union

**UNION RESULT**:告知开发人员，哪些表之间存在union查询

﻿

## 2.3 table

实际在哪张表中查询，这里的表可能是实际存在的表，也可能是临时表（如衍生表（derived）、union表等）

## 2.4 type：索引类型/类型

类型很多，这里只罗列常见常用的几个

system>const>eq_ref>ref>range>index>all（性能由高到低）

其中：system,const只是理想情况；实际能达到ref>range

（1）system:只有一条数据的系统表或衍生表只有一条数据的主查询

```
create table test01(tid int(3),tname varchar(20));
insert into test01 values(1,'a');
//增加索引
alter table test01 add constraint tid_pk primary key(tid);
explain select * from (select * from test01)t where tid = 1;
```

﻿

```
mysql> create table test01(tid int(3),tname varchar(20));
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> insert into test01 values(1,'a');
Query OK, 1 row affected (0.00 sec)

mysql> alter table test01 add constraint tid_pk primary key(tid);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from (select * from test01)t where tid = 1;
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test01 | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

﻿

（2）const:仅仅能查到一条数据的SQL，并且用于Primary key或union索引（类型与索引类型有关）

```
explain select tid from test01 where tid = 1;//const
```

验证：

```
alter table test01 drop primary key;
create index test01_index on test01(tid);/alter table test01 add index test01_index(tid);
explain select tid from test01 where tid = 1; //不是const
```

```
mysql> explain select tid from test01 where tid = 1;
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | test01 | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> alter table test01 drop primary key;
Query OK, 1 row affected (0.02 sec)
Records: 1  Duplicates: 0  Warnings: 0

mysql> create index test01_index on test01(tid);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select tid from test01 where tid = 1;
+----+-------------+--------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | test01 | NULL       | ref  | test01_index  | test01_index | 4       | const |    1 |   100.00 | Using index |
+----+-------------+--------+------------+------+---------------+--------------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

﻿



因为没有使用Primary key和Unique，所以不是const。

（3）eq_ref:唯一性索引：对于每个索引键的查询，返回匹配唯一行数据（有且只有1个，不能多、不能0）

常见于唯一索引和主键索引

验证：

用teacher和teacherCard两张表关联查询

在teacherCard表tcid(与teacher外键关联)上添加主键：

```
alter table teacherCard add constraint pk_tcid primary key(tcid);
```

由于外键在语法上允许重复，而eq_ref需要数据唯一，所以也给teacher中tcid加上唯一索引：

```
alter table teacher add constraint uk_tcid unique index(tcid);
```

```
explain select t.tcid from teacher t,teacherCard tc where t.tcid = tc.tcid;
```

```
mysql> alter table teacherCard add constraint pk_tcid primary key(tcid);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table teacher add constraint uk_tcid unique index(tcid);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select t.tcid from teacher t,teacherCard tc where t.tcid = tc.tcid;
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref          | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------------+
|  1 | SIMPLE      | tc    | NULL       | index  | PRIMARY       | PRIMARY | 4       | NULL         |    3 |   100.00 | Using index |
|  1 | SIMPLE      | t     | NULL       | eq_ref | uk_tcid       | uk_tcid | 5       | test.tc.tcid |    1 |   100.00 | Using index |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```



**（4）ref**:非唯一性索引，对于每个索引键的查询，**返回匹配的所有行（可以是0，多个）**

准备数据：

```
insert into teacher values(4,'tz',4);//为了有两个tz数据
insert into teacherCard values(4,'tzxxx');
```

测试：

```
alter table teacher add index index_name(tname);
explain select * from teacher where tname = 'tz';//type---ref
```

```
mysql> update teacher set tname = 'tz' where tid = 4;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> alter table teacher add index index_name(tname);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from teacher where tname = 'tz';
+----+-------------+---------+------------+------+---------------+------------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | teacher | NULL       | ref  | index_name    | index_name | 83      | const |    2 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```

**（5）range**:检索指定范围的行，where后面是一个范围查询（between,in,>,<,>=等）

```
alter table teacher add index tid_index(tid);

explain select t.* from teacher t where t.tid in (1,2);
//可能得到all(没有用到索引，因为in有时候会失效，从而没用到索引)
explain select t.* from teacher t where t.tid > 3;
//range
explain select t.* from teacher t where t.tid < 3;
//range
explain select t.* from teacher t where t.tid between 1 and 2;
//range
```

```
mysql> alter table teacher add index tid_index(tid);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select t.* from teacher t where t.tid in (1,2);
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | range | tid_index     | tid_index | 5       | NULL |    2 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

mysql>
mysql> explain select t.* from teacher t where t.tid > 3;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | range | tid_index     | tid_index | 5       | NULL |    3 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.01 sec)

mysql>
mysql> explain select t.* from teacher t where t.tid < 3;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | range | tid_index     | tid_index | 5       | NULL |    2 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

mysql>
mysql> explain select t.* from teacher t where t.tid between 1 and 2;
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | t     | NULL       | range | tid_index     | tid_index | 5       | NULL |    2 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

﻿



（6）index:查询全部索引中的数据

```
explain select tid from teacher;
```

//tid是索引，只需要扫描索引表，不需要所有表中的所有数据

（7）all:查询全部表中的数据

```
explain select cid from course;
```

//cid不是索引，需要全表扫描，即需要所有表中的所有数据

```
mysql> explain select tid from teacher;
+----+-------------+---------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | teacher | NULL       | index | NULL          | tid_index | 5       | NULL |    6 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+-----------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select cid from course;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | course | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```



小结：

system/const:结果只有一条数据

eq_ref:结果多条，但是每条数据是唯一的

ref:结果多条，但是每条数据是0条或多条

﻿

## 2.5 possible_key

可能用到的索引，是一种预测，不准。

﻿

## 2.6 key

实际用到的索引

—possible_key/key如果是null,则是说明没有用到索引

﻿

## 2.7 key_len:索引的长度

作用：

常用于判断复合索引是否被完全使用

```
create table test_k1(
 name char(20) not null default ''
);
//一个字段的情况
alter table test_k1 add index index_name(name);
explain select * from test_k1 where name = ''; 
//key_len:80
//在utf8:1个字符占4个字节

alter table test_k1 add column name1 char(20); 
//name1可以为null
alter table test_k1 add index index_name1(name1);
explain select * from test_k1 where name1 = '';
//key_len:81
//如果索引字段可以为null,则MySQL底层会使用1个字节用于标识name1可以为null。
```

```
mysql> create table test_k1(
    ->  name char(20) not null default ''
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> alter table test_k1 add index index_name(name);
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from test_k1 where name = '';
+----+-------------+---------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
| id | select_type | table   | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+---------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | test_k1 | NULL       | ref  | index_name    | index_name | 80      | const |    1 |   100.00 | Using where; Using index |
+----+-------------+---------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> alter table test_k1 add column name char(20);
ERROR 1060 (42S21): Duplicate column name 'name'
mysql> alter table test_k1 add column name1 char(20);
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql>
mysql> alter table test_k1 add index index_name1(name1);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from test_k1 where name1 = '';
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
| id | select_type | table   | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | test_k1 | NULL       | ref  | index_name1   | index_name1 | 81      | const |    1 |   100.00 | Using index condition |
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

﻿



```
//把两个索引都删了：
drop index index_name on test_k1;
drop index index_name1 on test_k1;
//增加一个复合索引
alter table test_k1 add index name_name1_index(name,name1);
explain select * from test_k1 where name1 = '';
//key_len:121---组合索引要用到第二个，那必然用到了前面的索引，所以：20*4+20*4+1=161
explain select * from test_k1 where name = '';
//key_len:40---用组合索引的第一个索引即可，所以：20*4 = 80
```

```
mysql> drop index index_name on test_k1;
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> drop index index_name1 on test_k1;
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table test_k1 add index name_name1_index(name,name1);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from test_k1 where name1 = '';
+----+-------------+---------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
| id | select_type | table   | partitions | type  | possible_keys    | key              | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+---------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | test_k1 | NULL       | index | name_name1_index | name_name1_index | 161     | NULL |    1 |   100.00 | Using where; Using index |
+----+-------------+---------+------------+-------+------------------+------------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from test_k1 where name = '';
+----+-------------+---------+------------+------+------------------+------------------+---------+-------+------+----------+--------------------------+
| id | select_type | table   | partitions | type | possible_keys    | key              | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+---------+------------+------+------------------+------------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | test_k1 | NULL       | ref  | name_name1_index | name_name1_index | 80      | const |    1 |   100.00 | Using where; Using index |
+----+-------------+---------+------------+------+------------------+------------------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

﻿

```
varchar(20)的情况:
alter table test_k1 add column name2 varchar(20);
//可以为null

alter table test_k1 add index name2_index(name2);
explain select * from test_k1 where name2 = '';
//key_len:63
//20*4 = 80 + 1(null) +2(用2个字节 标识可变长度---char是固定长度，而varchar是可变长度，要用两个字节标识) = 83
```

```
mysql> alter table test_k1 add column name2 varchar(20);
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql>
mysql> alter table test_k1 add index name2_index(name2);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from test_k1 where name2 = '';
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test_k1 | NULL       | ref  | name2_index   | name2_index | 83      | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

﻿

**utf8**:1个字符3个字节

**gbk**:1个字符2个字节

**latin**:1个字符1个字节



## 2.8 ref



注意与type中的ref区分

**作用**：

**指明当前表所参照的字段**

select…where a.c = b.x;(其中b.x可以是常量，则ref:const)

```
alter table course add index tid_index(tid);
explain select * from course c,teacher t where c.tid = t.tid and t.tname = 'tw';---检查两张表中的字段（这里用到的三个字段）是否有索引，没有的加上索引
```

```
mysql> alter table course add index tid_index(tid);
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from course c,teacher t where c.tid = t.tid and t.tname = 'tw';
+----+-------------+-------+------------+------+----------------------+------------+---------+-------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys        | key        | key_len | ref   | rows | filtered | Extra                                      |
+----+-------------+-------+------------+------+----------------------+------------+---------+-------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | t     | NULL       | ref  | index_name,tid_index | index_name | 83      | const |    1 |   100.00 | NULL                                       |
|  1 | SIMPLE      | c     | NULL       | ALL  | tid_index            | NULL       | NULL    | NULL  |    2 |   100.00 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+----------------------+------------+---------+-------+------+----------+--------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```

﻿



## 2.9 rows

被索引优化查询的数据个数（实际通过索引查到的数据个数）;

估算出结果集行数，表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数。

```
explain select * from course c,teacher t where c.tid = t.tid and t.tname = 'tz';
```

```
mysql> explain select * from course c,teacher t where c.tid = t.tid and t.tname = 'tz';
+----+-------------+-------+------------+------+----------------------+-----------+---------+------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys        | key       | key_len | ref        | rows | filtered | Extra       |
+----+-------------+-------+------------+------+----------------------+-----------+---------+------------+------+----------+-------------+
|  1 | SIMPLE      | c     | NULL       | ALL  | tid_index            | NULL      | NULL    | NULL       |    2 |   100.00 | Using where |
|  1 | SIMPLE      | t     | NULL       | ref  | index_name,tid_index | tid_index | 5       | test.c.tid |    1 |    33.33 | Using where |
+----+-------------+-------+------------+------+----------------------+-----------+---------+------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```

﻿





## 2.10 Extra

```
create table user (
id int primary key,
name varchar(20),
sex varchar(5),
index(name)
)engine=innodb;
 
insert into user values(1, 'shenjian','no');
insert into user values(2, 'zhangsan','no');
insert into user values(3, 'lisi', 'yes');
insert into user values(4, 'lisi', 'no');
```



**数据说明**：

　　用户表：id**主键索引**，name**普通索引**（非唯一），sex**无索引**；

　　四行记录：其中name普通索引存在重复记录lisi； 

**实验目的**：

通过构造各类SQL语句，对explain的Extra字段进行说明，启发式定位待优化低性能SQL语句。



###  **2.10.1 Using where**



**实验语句**：

```
explain select * from user where sex='no';
```

```
mysql> explain select * from user where sex='no';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |    25.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

﻿

**结果说明**：

　　Extra为Using where说明，SQL使用了where条件过滤数据。

 

需要注意的是：

（1）返回所有记录的SQL，不使用where条件过滤数据，大概率不符合预期，对于这类SQL往往需要进行优化；

（2）使用了where条件的SQL，并不代表不需要优化，往往需要配合explain结果中的type（连接类型）来综合判断；

 

　　本例虽然Extra字段说明使用了where条件过滤，但type属性是ALL，表示需要扫描全部数据，仍有优化空间。

　　常见的优化方法为，在where过滤属性上添加索引。

*注意**：**本例中，sex字段区分度不高，添加索引对性能提升有限。*



### **2.10.2** Using index

**实验语句**：

```
explain select id,name from user where name='shenjian';
```

```
mysql> explain select id,name from user where name='shenjian';
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | ref  | name          | name | 83      | const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

**结果说明**：

　　Extra为Using index说明，SQL所需要返回的所有列数据均在一棵索引树上，而无需访问实际的行记录。

　　这类SQL语句往往性能较好。

**问题来了，什么样的列数据，会包含在索引树上呢？**



### **2.10.3** **Using index condition**

```
explain select id,name,sex from user where name='shenjian';
```

画外音：该SQL语句与上一个SQL语句不同的地方在于，被查询的列，多了一个sex字段。

```
mysql> explain select id,name,sex from user where name='shenjian';
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user  | NULL       | ref  | name          | name | 83      | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```

﻿

**结果说明：**

Extra为Using index condition说明，确实命中了索引，但不是所有的列数据都在索引树上，还需要访问实际的行记录。

*注意：**聚集索引，普通索引的底层实现差异**。*

这类SQL语句性能也较高，但不如Using index。

### **2.10.4** **Using filesort**

```
explain select * from user order by sex;
```

```
mysql> explain select * from user order by sex;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
```

**结果说明：**

　　Extra为Using filesort说明，得到所需结果集，需要对所有记录进行文件排序。

　　这类SQL语句性能极差，需要进行优化。

　　典型的，在一个没有建立索引的列上进行了order by，就会触发filesort，常见的优化方案是，在order by的列上添加索引，避免每次查询都全量排序。



### **2.10.5** **Using temporary**

```
explain select name from user group by name order by sex;
```

```
mysql> explain select * from user group by sex order by name;
ERROR 1055 (42000): Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'test.user.id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
mysql>
mysql> SHOW VARIABLES LIKE '%sql_mode%';
+---------------+-----------------------------------------------------------------------------------------------------------------------+
| Variable_name | Value                                                                                                                 |
+---------------+-----------------------------------------------------------------------------------------------------------------------+
| sql_mode      | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION |
+---------------+-----------------------------------------------------------------------------------------------------------------------+
1 row in set (0.02 sec)

mysql>
mysql> set sql_mode = '';
Query OK, 0 rows affected (0.00 sec)

mysql> set sql_mode = 'NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql>
mysql> explain select * from user group by sex order by name;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    4 |   100.00 | Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)
```

**结果说明：**

　　Extra为Using temporary说明，需要建立临时表(temporary table)来暂存中间结果。 

　　这类SQL语句性能较低，往往也需要进行优化。

　　典型的，group by和order by同时存在，且作用于不同的字段时，就会建立临时表，以便计算出最终的结果集。

﻿

### **2.10.6** **Using join buffer (Block Nested Loop)**

```
explain select * from user where id in(select id from user where sex='no');
```

```
mysql> explain select * from user where id in(select id from user where sex='no');
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref          | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL         |    4 |    25.00 | Using where |
|  1 | SIMPLE      | user  | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | test.user.id |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```



**结果说明：**

　　Extra为Using join buffer (Block Nested Loop)说明，需要进行嵌套循环计算。

　　*注意：内层和外层的type均为ALL，rows均为4，需要循环进行4\*4次计算。*

 

　　这类SQL语句性能往往也较低，需要进行优化。

　　典型的，两个关联表join，关联字段均未建立索引，就会出现这种情况。常见的优化方案是，在关联字段上添加索引，避免每次嵌套循环计算。



结论：

1、Extra中的为Using index的情况:

　   where筛选列是索引的前导列 &&查询列被索引覆盖 && where筛选条件是一个基于索引前导列的查询，意味着通过索引超找就能直接找到符合条件的数据，并且无须回表

2、Extra中的为空的情况:

　　查询列存在未被索引覆盖&&where筛选列是索引的前导列，意味着通过索引超找并且通过“回表”来找到未被索引覆盖的字段，

3、Extra中的为Using where Using index：

　　出现Using where Using index意味着是通过索引扫描（或者表扫描）来实现sql语句执行的，即便是索引前导列的索引范围查找也有一点范围扫描的动作，不管是前非索引前导列引起的，还是非索引列查询引起的。
