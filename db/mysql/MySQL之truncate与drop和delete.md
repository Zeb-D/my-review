### MySQL之truncate、delete、drop对表操作分析

参考链接：https://www.cnblogs.com/SaraMoring/p/5607537.html



#### 相同点：

- truncate和不带where子句的delete、以及drop都会删除表内的数据。
- drop、truncate都是DDL语句(数据定义语言),执行后会自动提交。



#### 不同点：

-  truncate 和 delete 只删除数据不删除表的结构(定义)
  drop 语句将删除表的结构被依赖的约束(constrain)、触发器(trigger)、索引(index)；依赖于该表的存储过程/函数将保留,但是变为 invalid 状态。

  ​

- delete 语句是数据库操作语言(dml)，这个操作会放到 rollback segement 中，事务提交之后才生效；如果有相应的 trigger，执行的时候将被触发。
  truncate、drop 是数据库定义语言(ddl)，操作立即生效，原数据不放到 rollback segment 中，不能回滚，操作不触发 trigger。

  ​

- delete 语句不影响表所占用的 extent，高水线(high watermark)保持原位置不动
  drop 语句将表所占用的空间全部释放。

  truncate 语句缺省情况下见空间释放到 minextents个 extent，除非使用reuse storage；truncate 会将高水线复位(回到最开始)。

  ​

- 速度，一般来说: drop> truncate > delete

  ​

- 安全性：小心使用 drop 和 truncate，尤其没有备份的时候.否则哭都来不及
  使用上,想删除部分数据行用 delete，注意带上where子句. 回滚段要足够大.
  想删除表,当然用 drop
  想保留表而将所有数据删除，如果和事务无关，用truncate即可。如果和事务有关,或者想触发trigger,还是用delete。
  如果是整理表内部的碎片，可以用truncate跟上reuse stroage，再重新导入/插入数据。

  ​

- delete是DML语句,不会自动提交。drop/truncate都是DDL语句,执行后会自动提交。

  ​

- TRUNCATE   TABLE   在功能上与不带   WHERE   子句的   DELETE   语句相同：二者均删除表中的全部行。但   TRUNCATE   TABLE   比   DELETE   速度快，且使用的系统和事务日志资源少。DELETE   语句每次删除一行，并在事务日志中为所删除的每行记录一项。TRUNCATE   TABLE   通过释放存储表数据所用的数据页来删除数据，并且只在事务日志中记录页的释放。 

  ​

- TRUNCATE   TABLE   删除表中的所有行，但表结构及其列、约束、索引等保持不变。新行标识所用的计数值重置为该列的种子。如果想保留标识计数值，请改用   DELETE。如果要删除表定义及其数据，请使用   DROP   TABLE   语句。

  ​

- 对于由   FOREIGN   KEY   约束引用的表，不能使用   TRUNCATE   TABLE，而应使用不带   WHERE   子句的   DELETE   语句。由于   TRUNCATE   TABLE   不记录在日志中，所以它不能激活触发器。

  ​

- TRUNCATE   TABLE   不能用于参与了索引视图的表。  

  ​

   

