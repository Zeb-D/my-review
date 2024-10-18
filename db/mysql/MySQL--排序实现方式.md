本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------



### 背景

在职业生涯中，关于mysql order by线上的问题有一些，其中关键的点就是 explain sql出现的`filesort` （被几个半桶水说成有慢sql问题），还有在某些老业务mysql低版本出现的数据重复问题。再加上自己对这块模棱两可就深入下总结；



### 概念

排序是数据库中的一个基本功能，MySQL也不例外。用户通过Order by语句即能达到将指定的结果集排序的目的，其实不仅仅是Order by语句，Group by语句，Distinct语句都会隐含使用排序。

> 本文首先会简单介绍SQL如何利用索引避免排序代价，
>
> 然后会介绍MySQL实现排序的内部原理，并介绍与排序相关的参数，
>
> 最后会给出几个“奇怪”排序例子，来谈谈排序一致性问题，并说明产生现象的本质原因。



### 排序优化与索引使用

​    为了优化SQL语句的排序性能，最好的情况是避免排序，合理利用索引是一个不错的方法。因为索引本身也是有序的，如果在需要排序的字段上面建立了合适的索引，那么就可以跳过排序的过程，提高SQL的查询速度。

下面我通过一些典型的SQL来说明哪些SQL可以利用索引减少排序，哪些SQL不能。



假设t1表存在索引key1(key_part1,key_part2),key2(key2)

- a.可以利用索引避免排序的SQL

```
SELECT * FROM t1 ORDER BY key_part1,key_part2;

SELECT * FROM t1 WHERE key_part1 = constant ORDER BY key_part2;

SELECT * FROM t1 WHERE key_part1 > constant ORDER BY key_part1 ASC;

SELECT * FROM t1 WHERE key_part1 = constant1 AND key_part2 > constant2 ORDER BY key_part2;
```

- b.不能利用索引避免排序的SQL

```
//排序字段在多个索引中，无法使用索引排序
SELECT * FROM t1 ORDER BY key_part1,key_part2, key2;

//排序键顺序与索引中列顺序不一致，无法使用索引排序
SELECT * FROM t1 ORDER BY key_part2, key_part1;

//升降序不一致，无法使用索引排序
SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC;

//key_part1是范围查询，key_part2无法使用索引排序
SELECT * FROM t1 WHERE key_part1> constant ORDER BY key_part2;
```



### 排序实现的算法

​     对于不能利用索引避免排序的SQL，数据库不得不自己实现排序功能以满足用户需求，此时SQL的执行计划中会出现“Using filesort”，这里需要注意的是filesort并不意味着就是文件排序，其实也有可能是内存排序，这个主要由sort_buffer_size参数与结果集大小确定。



MySQL内部实现排序主要有3种方式，常规排序、优化排序和优先队列排序，

主要涉及3种排序算法：快速排序、归并排序和堆排序。



假设表结构和SQL语句如下：

```
CREATE TABLE t1(id int, col1 varchar(64), col2 varchar(64), col3 varchar(64), PRIMARY KEY(id),key(col1,col2));

SELECT col1,col2,col3 FROM t1 WHERE col1>100 ORDER BY col2;
```



#### 常规排序

(1).从表t1中获取满足WHERE条件的记录

(2).对于每条记录，将记录的主键+排序键(id,col2)取出放入sort buffer

(3).如果sort buffer可以存放所有满足条件的(id,col2)对，则进行排序；否则sort buffer满后，进行排序并固化到临时文件中。(排序算法采用的是快速排序算法)

(4).若排序中产生了临时文件，需要利用归并排序算法，保证临时文件中记录是有序的

(5).循环执行上述过程，直到所有满足条件的记录全部参与排序

(6).扫描排好序的(id,col2)对，并利用id去捞取SELECT需要返回的列(col1,col2,col3)

(7).将获取的结果集返回给用户。



​      从上述流程来看，是否使用文件排序主要看sort buffer是否能容下需要排序的(id,col2)对，这个buffer的大小由s`ort_buffer_size`参数控制。

此外一次排序需要两次IO，一次是捞(id,col2),第二次是捞(col1,col2,col3)，由于返回的结果集是按col2排序，因此id是乱序的，通过乱序的id去捞(col1,col2,col3)时会产生大量的随机IO。

对于第二次MySQL本身一个优化，即在捞之前首先将id排序，并放入缓冲区，这个缓存区大小由参数`read_rnd_buffer_size`控制，然后有序去捞记录，将随机IO转为顺序IO。



#### 优化排序

​     常规排序方式除了排序本身，还需要额外两次IO。优化的排序方式相对于常规排序，减少了第二次IO。主要区别在于，放入sort buffer不是(id,col2),而是(col1,col2,col3)。由于sort buffer中包含了查询需要的所有字段，因此排序完成后可以直接返回，无需二次捞数据。这种方式的代价在于，同样大小的sort buffer，能存放的(col1,col2,col3)数目要小于(id,col2)，如果sort buffer不够大，可能导致需要写临时文件，造成额外的IO。当然MySQL提供了参数max_length_for_sort_data，只有当排序元组小于max_length_for_sort_data时，才能利用优化排序方式，否则只能用常规排序方式。



#### 优先队列排序

​     为了得到最终的排序结果，无论怎样，我们都需要将所有满足条件的记录进行排序才能返回。那么相对于优化排序方式，是否还有优化空间呢？5.6版本针对Order by limit M，N语句，在空间层面做了优化，加入了一种新的排序方式--优先队列，这种方式采用堆排序实现。

堆排序算法特征正好可以解limit M，N 这类排序的问题，虽然仍然需要所有元素参与排序，但是只需要M+N个元组的sort buffer空间即可，对于M，N很小的场景，基本不会因为sort buffer不够而导致需要临时文件进行归并排序的问题。对于升序，采用大顶堆，最终堆中的元素组成了最小的N个元素，对于降序，采用小顶堆，最终堆中的元素组成了最大的N的元素。



### 线上问题案例

#### 环境准备

环境准备（我是单独在ec2部署的）(本地电脑跳过)：

> yum remove docker \
>                    docker-client \
>                    docker-client-latest \
>                    docker-common \
>                    docker-latest \
>                    docker-latest-logrotate \
>                    docker-logrotate \
>                    docker-engine
>                    
>  yum-config-manager \
>      --add-repo \
>      https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo   
>      
> sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin

配置国内镜像源(本地电脑跳过)：

> [root@iZwz9hxarfzj17lx1c3ameZ ~]# sudo tee /etc/docker/daemon.json <<-'EOF'
> > {
> > "registry-mirrors": ["https://3tgljxkm.mirror.aliyuncs.com"]
> > }
> > EOF
> > {
> >   "registry-mirrors": ["https://3tgljxkm.mirror.aliyuncs.com"]
> > }
> >
> > [root@iZwz9hxarfzj17lx1c3ameZ ~]# sudo systemctl daemon-reload
> > [root@iZwz9hxarfzj17lx1c3ameZ ~]# sudo systemctl restart docker
> > [root@iZwz9hxarfzj17lx1c3ameZ ~]# 
> > [root@iZwz9hxarfzj17lx1c3ameZ ~]# docker info

// 启动测试mysql

```
docker run -p 3306:3306 --privileged=true -v ~/docker/mysql/test:/var/lib/mysql:rw --name ydMysql -e MYSQL_ROOT_PASSWORD=yd_mysql -d mysql:5.6
```

// 创建测试库：

> [root@iZwz9hxarfzj17lx1c3ameZ ~]# sudo docker exec -it ydMysql /bin/bash
> root@2864a8307e2b:/# mysql -uroot -pyd_mysql
> mysql> CREATE DATABASE IF NOT EXISTS test DEFAULT CHARACTER SET utf8mb4
>     -> ;
> Query OK, 1 row affected (0.00 sec)
>
> mysql> use test;
> Database changed



#### Mysql从5.5迁移到5.6以后，发现分页出现了重复值

测试表与数据准备：

> mysql> create table t1(id int primary key, c1 int, c2 varchar(128));
> Query OK, 0 rows affected (0.01 sec)
>
> mysql> insert into t1 values(1,1,'a');
> Query OK, 1 row affected (0.01 sec)
>
> mysql> insert into t1 values(2,2,'b');
> Query OK, 1 row affected (0.00 sec)
>
> mysql> insert into t1 values(3,2,'c');
> Query OK, 1 row affected (0.00 sec)
>
> mysql> insert into t1 values(4,2,'d');
> Query OK, 1 row affected (0.01 sec)
>
> mysql> insert into t1 values(5,3,'e');
> Query OK, 1 row affected (0.00 sec)
>
> mysql> insert into t1 values(6,4,'f');
> Query OK, 1 row affected (0.00 sec)
>
> mysql> insert into t1 values(7,5,'g');
> Query OK, 1 row affected (0.00 sec)

// 分页查询重复

![mysql-orderby-impl-qa1.png](../../image/mysql-orderby-impl-qa1.png)



现象分析：

我们可以看到 id为4的这条记录居然同时出现在两次查询中，这明显是不符合预期的，而且在5.5版本中没有这个问题。

产生这个现象的原因就是5.6针对limit M,N的语句采用了优先队列，而优先队列采用堆实现，

比如上述的例子order by c1 asc limit 0，3 需要采用大小为3的大顶堆；limit 3，3需要采用大小为6的大顶堆。

由于c1为2的记录有3条，而堆排序是非稳定的(对于相同的key值，无法保证排序后与排序前的位置一致)，所以导致分页重复的现象。

为了避免这个问题，我们可以在排序中加上唯一值，比如主键id，这样由于id是唯一的，确保参与排序的key值不相同。将SQL写成如下：

> select * from t1 order by c1,id asc limit 0,3;
> select * from t1 order by c1,id asc limit 3,3;



#### 两个类似的查询语句，除了返回列不同，其它都相同，但排序的结果不一致。

测试表与数据：

> create table t2(id int primary key, status int, c1 varchar(255),c2 varchar(255),c3 varchar(255),key(c1));
> insert into t2 values(7,1,'a',repeat('b',255),repeat('c',255));
> insert into t2 values(6,2,'b',repeat('b',255),repeat('c',255));
> insert into t2 values(5,2,'c',repeat('b',255),repeat('c',255));
> insert into t2 values(4,2,'a',repeat('b',255),repeat('c',255));
> insert into t2 values(3,3,'b',repeat('b',255),repeat('c',255));
> insert into t2 values(2,4,'c',repeat('b',255),repeat('c',255));
> insert into t2 values(1,5,'a',repeat('b',255),repeat('c',255));

分别执行SQL语句：

> select id,status,c1,c2 from t2 force index(c1) where c1>='b' order by status;
> select id,status from t2 force index(c1) where c1>='b' order by status;



执行结果如下：

![mysql-orderby-impl-qa2.png](../../image/mysql-orderby-impl-qa2.png)

> 本来select id,status from t2 force index(c1) where c1>='b' order by status;
>
> 应该是 5 和 6 是反的，当时线上DMP发生了这种，当时忘记了截图，后面重试也不行。
>
> mysql> select id,status from t2 force index(c1) where c1>='b' order by status;
> +----+--------+
> | id | status |
> +----+--------+
> |  6 |      2 |
> |  5 |      2 |
> |  3 |      3 |
> |  2 |      4 |
> +----+--------+
> 4 rows in set (0.00 sec)



看看两者的执行计划是否相同：

> explain select id,status,c1,c2 from t2 force index(c1) where c1>='b' order by status;
>
> explain select id,status from t2 force index(c1) where c1>='b' order by status;

> mysql> explain select id,status,c1,c2 from t2 force index(c1) where c1>='b' order by status;
> +----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------+
> | id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                       |
> +----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------+
> |  1 | SIMPLE      | t2    | range | c1            | c1   | 767     | NULL |    4 | Using where; Using filesort |
> +----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------+
> 1 row in set (0.00 sec)
>
> mysql> explain select id,status from t2 force index(c1) where c1>='b' order by status;
> +----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------+
> | id | select_type | table | type  | possible_keys | key  | key_len | ref  | rows | Extra                       |
> +----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------+
> |  1 | SIMPLE      | t2    | range | c1            | c1   | 767     | NULL |    4 | Using where; Using filesort |
> +----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------+
> 1 row in set (0.00 sec)



分析：

为了说明问题，我在语句中加了force index的hint，确保能走上c1列索引。语句通过c1列索引捞取id，然后去表中捞取返回的列。

根据c1列值的大小，记录在c1索引中的相对位置如下：

(c1,id)===(b,6),(b,3),(5,c),(c,2)，对应的status值分别为2 3 2 4。

从表中捞取数据并按status排序，则相对位置变为(6,2,b),(5,2,c),(3,3,c),(2,4,c)，这就是第二条语句查询返回的结果，那么为什么第一条查询语句(6,2,b),(5,2,c)是调换顺序的呢？

这里要看上面提到的**常规排序和优化排序**中的部分，就可以明白原因了。

由于第一条查询返回的列的字节数超过了max_length_for_sort_data，导致排序采用的是常规排序，而在这种情况下MYSQL将rowid排序，将随机IO转为顺序IO，所以返回的是5在前，6在后；而第二条查询采用的是优化排序，没有第二次捞取数据的过程，保持了排序后记录的相对位置。

对于第一条语句，若想采用优化排序，我们将max_length_for_sort_data设置调大即可，比如2048。





### 留几个问题

*可以通过公众号一起交流／留言。*

Q1：排序实现的地方是在MySQL Server 还是存储层实现的？



Q2：你认为最新版本如 `>= MySQL 8.0` 会有上面案例出现低级问题吗？为什么？



Q3：对于问题2 `两个类似的查询语句，除了返回列不同，其它都相同，但排序的结果不一致` ，大家有碰到过吗？

