本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### Mysql之MYISAM、INODB的区别



### MySQL存储引擎背景：

数据库中的存储引擎其实是对使用了该引擎的表进行某种设置，数据库中的表设定了什么存储引擎，那么该表在数据存储方式、数据更新方式、数据查询性能以及是否支持索引等方面就会有不同的“效果”。在MySQL数据库中存在着多种引擎（不同版本的MySQL数据库支持的引擎不同）。



一般来说，MySQL有以下几种引擎：ISAM、MyISAM、HEAP（也称为MEMORY）、CSV、BLACKHOLE、ARCHIVE、PERFORMANCE_SCHEMA、InnoDB、 Berkeley、Merge、Federated和Cluster/NDB等，除此以外我们也可以参照MySQL++ API创建自己的数据库引擎。



###MySQL存储引擎：

- **ISAM**
  ​        该引擎在读取数据方面速度很快，而且不占用大量的内存和存储资源；但是ISAM不支持事务处理、不支持外来键、不能够容错、也不支持索引。该引擎在包括MySQL 5.1及其以上版本的数据库中不再支持。

- **MyISAM**
  ​        该引擎基于ISAM数据库引擎，除了提供ISAM里所没有的索引和字段管理等大量功能，MyISAM还使用一种表格锁定的机制来优化多个并发的读写操作，但是需要经常运行OPTIMIZE TABLE命令，来恢复被更新机制所浪费的空间，否则碎片也会随之增加，最终影响数据访问性能。MyISAM还有一些有用的扩展，例如用来修复数据库文件的MyISAMChk工具和用来恢复浪费空间的 MyISAMPack工具。MyISAM强调了快速读取操作，主要用于高负载的select，这可能也是MySQL深受Web开发的主要原因：在Web开发中进行的大量数据操作都是读取操作，所以大多数虚拟主机提供商和Internet平台提供商（Internet Presence Provider，IPP）只允许使用MyISAM格式。
  MyISAM类型的表支持三种不同的存储结构：静态型、动态型、压缩型。

  静态型：指定义的表列的大小是固定（即不含有：xblob、xtext、varchar等长度可变的数据类型），这样MySQL就会自动使用静态MyISAM格式。使用静态格式的表的性能比较高，因为在维护和访问以预定格式存储数据时需要的开销很低；但这种高性能是以空间为代价换来的，因为在定义的时候是固定的，所以不管列中的值有多大，都会以最大值为准，占据了整个空间。
  ​        动态型：如果列（即使只有一列）定义为动态的（xblob, xtext, varchar等数据类型），这时MyISAM就自动使用动态型，虽然动态型的表占用了比静态型表较少的空间，但带来了性能的降低，因为如果某个字段的内容发生改变则其位置很可能需要移动，这样就会导致碎片的产生，随着数据变化的增多，碎片也随之增加，数据访问性能会随之降低。
  ​        对于因碎片增加而降低数据访问性这个问题，有两种解决办法：
  ​        a、尽可能使用静态数据类型；
  ​        b、经常使用optimize table table_name语句整理表的碎片，恢复由于表数据的更新和删除导致的空间丢失。如果存储引擎不支持 optimize table table_name则可以转储并        重新加载数据，这样也可以减少碎片；
  ​        压缩型：如果在数据库中创建在整个生命周期内只读的表，则应该使用MyISAM的压缩型表来减少空间的占用。

- **HEAP**（也称为MEMORY）
  ​        该存储引擎通过在内存中创建临时表来存储数据。每个基于该存储引擎的表实际对应一个磁盘文件，该文件的文件名和表名是相同的，类型为.frm。该磁盘文件只存储表的结构，而其数据存储在内存中，所以使用该种引擎的表拥有极高的插入、更新和查询效率。这种存储引擎默认使用哈希（HASH）索引，其速度比使用B-+Tree型要快，但也可以使用B树型索引。由于这种存储引擎所存储的数据保存在内存中，所以其保存的数据具有不稳定性，比如如果mysqld进程发生异常、重启或计算机关机等等都会造成这些数据的消失，所以这种存储引擎中的表的生命周期很短，一般只使用一次。

- **CSV**（Comma-Separated Values逗号分隔值）
  ​        使用该引擎的MySQL数据库表会在MySQL安装目录data文件夹中的和该表所在数据库名相同的目录中生成一个.CSV文件（所以，它可以将CSV类型的文件当做表进行处理），这种文件是一种普通文本文件，每个数据行占用一个文本行。该种类型的存储引擎不支持索引，即使用该种类型的表没有主键列；另外也不允许表中的字段为null。

- **BLACKHOLE**（黑洞引擎）
  ​        该存储引擎支持事务，而且支持mvcc的行级锁，写入这种引擎表中的任何数据都会消失，主要用于做日志记录或同步归档的中继存储，这个存储引擎除非有特别目的，否则不适合使用。详见博客《[BlackHole 存储引擎](http://blog.csdn.net/gaohuanjie/article/details/50923388)》

- **ARCHIVE**
  ​        该存储引擎非常适合存储大量独立的、作为历史记录的数据。区别于InnoDB和MyISAM这两种引擎，ARCHIVE提供了压缩功能，拥有高效的插入速度，但是这种引擎不支持索引，所以查询性能较差一些。

- **PERFORMANCE_SCHEMA**
  ​        该引擎主要用于收集数据库服务器性能参数。这种引擎提供以下功能：提供进程等待的详细信息，包括锁、互斥变量、文件信息；保存历史的事件汇总信息，为提供MySQL服务器性能做出详细的判断；对于新增和删除监控事件点都非常容易，并可以随意改变mysql服务器的监控周期，例如（CYCLE、MICROSECOND）。

- **InnoDB**
  ​        该存储引擎为MySQL表提供了ACID事务支持、系统崩溃修复能力和多版本并发控制（即MVCC Multi-Version Concurrency Control）的行级锁;该引擎支持自增长列（auto_increment）,自增长列的值不能为空，如果在使用的时候为空则自动从现有值开始增值，如果有但是比现在的还大，则直接保存这个值; 该引擎存储引擎支持外键（foreign key） ,外键所在的表称为子表而所依赖的表称为父表。该引擎在5.5后的MySQL数据库中为默认存储引擎。

- Merge
  ​        该引擎将一定数量的MyISAM表联合而成一个整体。参见博客《[MySQL Merge存储引擎](http://blog.csdn.net/gaohuanjie/article/details/50947055)》

- Federated
  ​        该存储引擎可以不同的Mysql服务器联合起来，逻辑上组成一个完整的数据库。这种存储引擎非常适合数据库分布式应用。

- Cluster/NDB
  ​        该存储引擎用于多台数据机器联合提供服务以提高整体性能和安全性。适合数据量大、安全和性能要求高的场景。





### MyISAM和InnoDB存储引擎的比较

> MyIASM和Innodb两种引擎所使用的索引的数据结构是什么？

> 答案:都是B+树!

MyIASM引擎，B+树的数据结构中存储的内容实际上是实际数据的地址值。也就是说它的索引和实际数据是分开的，**只不过使用索引指向了实际数据。这种索引的模式被称为非聚集索引。**

Innodb引擎的索引的数据结构也是B+树，**只不过数据结构中存储的都是实际的数据，这种索引有被称为聚集索引**。



##### MyISAM

MyISAM是基于传统的ISAM类型，支持全文搜索，但不是事务安全的，而且不支持外键。每张MyISAM表存放在三个文件中：frm 文件存放表格定义；数据文件是MYD (MYData)；索引文件是MYI (MYIndex)。

##### InnoDB

InnoDB是事务型引擎，支持回滚、崩溃恢复能力、多版本并发控制、ACID事务，支持行级锁定（InnoDB表的行锁不是绝对的，如果在执行一个SQL语句时MySQL不能确定要扫描的范围，InnoDB表同样会锁全表，如like操作时的SQL语句），以及提供与Oracle类型一致的不加锁读取方式。InnoDB存储它的表和索引在一个表空间中，表空间可以包含数个文件。



### 主要区别：

- MyISAM是非事务安全型的，而InnoDB是事务安全型的。
- MyISAM锁的粒度是表级，而InnoDB支持行级锁定。
- MyISAM支持全文类型索引，而InnoDB不支持全文索引。
- MyISAM相对简单，所以在效率上要优于InnoDB，小型应用可以考虑使用MyISAM。
- MyISAM表是保存成文件的形式，在跨平台的数据转移中使用MyISAM存储会省去不少的麻烦。
- InnoDB表比MyISAM表更安全，可以在保证数据不会丢失的情况下，切换非事务表到事务表（alter table tablename type=innodb）。

### 应用场景：

- MyISAM管理非事务表。它提供高速存储和检索，以及全文搜索能力。如果应用中需要执行大量的SELECT查询，那么MyISAM是更好的选择。
- InnoDB用于事务处理应用程序，具有众多特性，包括ACID事务支持。如果应用中需要执行大量的INSERT或UPDATE操作，则应该使用InnoDB，这样可以提高多用户并发操作的性能。
- myisam只缓存索引，不缓存数据，文件系统也会缓存。



### 常用命令：

（1）查看表的存储类型（三种）：

- show create table tablename
- show table status from  dbname  where name=tablename
- mysqlshow  -u user -p password --status dbname tablename

（2）修改表的存储引擎：

- alter table tablename engine=InnoDB

（3）启动mysql数据库的命令行中添加以下参数使新发布的表都默认使用事务：

- --default-table-engine=InnoDB


（4）数据库

   show databases; --查看数据库

   use dbNmae;使用某数据库

  show tables;  展示所有表

（5）表的列

describe tableName；获取有关列的信息




