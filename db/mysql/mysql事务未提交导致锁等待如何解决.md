# mysql事务未提交导致锁等待如何解决



<br>

### 查看当前连接Id（线程Id）

`select` `connection_id();`

<br>

### 可以通过查看表information_schema.innodb_lock，获取锁的状态

`select * from information_schema.innodb_locks;`

<br>

### 查看系统参数锁超时时间 innodb_lock_wait_timeout

`show variables like innodb_lock_wait_timeout;`

<br>

### 查看未提交的事务

`SELECT * from information_schema.INNODB_TRX`

<br>

### 获取事物的描述

`show engine innodb status\G`

主要看TRANSACTIONS 这部分

<br>

## 如何解决未提交的事务

1）如果能知道哪个用户在执行这个操作，让他提交一下（这种可能性很小）

2）kill掉这个线程id号，让事务回滚

​	`show processlist;`

​	获取所有处理线程，找到相对应引起死锁的线程Id

​	`kill Id`

