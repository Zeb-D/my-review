# replace into 和 insert into … on duplicate key update区别

## 引发问题

一张表的结构如下  

```sql
CREATE TABLE `CheckinRecord` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `loversID` int(11) NOT NULL DEFAULT '0' COMMENT '情侣ID,分表键',
  `memberID` bigint(20) NOT NULL DEFAULT '0' COMMENT '用户ID',
  `objectID` bigint(20) NOT NULL DEFAULT '0' COMMENT '对象ID',
  `checkinDate` date NOT NULL COMMENT '签到日期',
  `continuousDays` int(11) NOT NULL DEFAULT '1' COMMENT '连续签到天数',
  `createTime` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '记录创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `unq_loversid_checkindate` (`loversID`,`checkinDate`),
  KEY `idx_create_time` (`createTime`)
) ENGINE=InnoDB AUTO_INCREMENT=312 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
```

业务通过使用insert into … on duplicate key返回的affected rows = 1 or 2来判断是插入还是更新以便接下来的判断。
sql如下

```sql
insert into CheckinRecord (loversID,memberID,objectID,checkinDate,continuousDays,createTime)
		value (#{loversID},#{memberID},#{objectID},#{checkinDate},#{continuousDays},#{createTime})
		on duplicate key update createTime = now()
```

正常情况下 affected rows = 1 是可以判断出来插入， affected rows = 2 是可以判断出来更新的，但是由于更新的createTime是秒级别的，所以出现秒并发的情况，实际上更新操作返回的是1而不是2。但是使用replace into就永远不会出现这种问题。
我们比较一下两者异同点

### 相同点

1. 都是通过查询主键或者唯一索引来实现插入更新的功能
2. 当主键/唯一索引不存在时，两者等同于insert into， affected rows = 1

## 不同点

1. 当key存在冲突时，replace into 会直接删除旧记录，插入一条新记录。新纪录的key不会改变，其他的字段如果没设置会使用默认值，affected rows = 2。 如果key是主键冲突，auto_increment是不会更改，但如果是唯一索引冲突，auto_increment + 1； insert into ... on duplicate key update 在key冲突的时候也会删除旧的记录，但是会拷贝原来的值，并且更新设置的字段，affected rows = 2，auto_increment的变化同replace into相同，但是如果更新的值和现存在的值相同，那么affected rows = 0，意味着不会有任何操作（但是如果mysql中CLIENT_FOUND_ROWS设置成了mysql_real_connect(), 那么返回的就是1)

> With ON DUPLICATE KEY UPDATE, the affected-rows value per row is 1 if the row is inserted as a new row, 2 if an existing row is updated, and 0 if an existing row is set to its current values. If you specify the CLIENT_FOUND_ROWS flag to the mysql_real_connect() C API function when connecting to mysqld, the affected-rows value is 1 (not 0) if an existing row is set to its current values.  


## 引用文档

[MySQL :: MySQL 5.7 Reference Manual :: 13.2.5.2 INSERT … ON DUPLICATE KEY UPDATE Syntax](https://dev.mysql.com/doc/refman/5.7/en/insert-on-duplicate.html)