### 预备

```
mkdir -p /data/docker/
```

列表出无效容器

```
docker ps -a|grep Exited|awk '{print $1}'
```

批量删除无效容器

```
docker rm `docker ps -a|grep Exited|awk '{print $1}'
```

`



### redis

```
mkdir -p /data/docker/redis/data
mkdir -p /data/docker/redis/conf
```

 

创建redis

```
docker run -p 6379:6379 -v /data/docker/redis/data:/data:rw --name ydRedis -d redis:4.0.8 redis-server --appendonly yes --protected-mode no --requirepass "yd_redis"
```

测试

```
sudo docker exec -it ydRedis /bin/bash
redis-cli -a yd_redis
```

或者

```
telnet 127.0.0.1 6379
auth ydRedis
```



### mysql

```
mkdir -p /data/docker/mysql/data
mkdir -p /data/docker/mysql/conf
```

创建mysql

```
docker run -p 3306:3306 -v /data/docker/mysql/data:/var/lib/mysql:rw --name ydMysql -e MYSQL_ROOT_PASSWORD=yd_mysql -d mysql:8.0
```

测试

```
sudo docker exec -it ydMysql /bin/bash
mysql -uroot -pyd_mysql
```

创建用户

```
CREATE USER 'yd'@'%' IDENTIFIED BY 'yd@Test';
GRANT ALL PRIVILEGES ON  test.* TO 'yd'@'%';
FLUSH PRIVILEGES;
```

退出：

```
mysql -uyd -pyd@Test
select User, Host from mysql.user;
```

创建数据库

```
CREATE DATABASE IF NOT EXISTS test DEFAULT CHARACTER SET utf8mb4
```



### zookeeper

```
docker run -p 2181:2181 -v /data/docker/zk/zkdata:/data:rw -v /data/docker/zk/zklogdata:/datalog --name ydZk -d zookeeper:3.4
```



### kafka

```
docker run -d --name ydKafka -v /data/kafka:/kafka:rw --publish 9092:9092 --link ydZk --env KAFKA_ZOOKEEPER_CONNECT=ydZk:2181 --env KAFKA_ADVERTISED_HOST_NAME=localhost --env KAFKA_ADVERTISED_PORT=9092 wurstmeister/kafka:2.11-0.11.0.3
```

```
docker exec ydZk bin/zkCli.sh ls /brokers/ids
```

创建topic

```
bin/kafka-topics.sh --if-not-exists --create --zookeeper ydZk:2181 --replication-factor 1 --partitions 1 --topic ydkafka
```

发送消息

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic ydkafka
```

消费消息

```
bin/kafka-console-consumer.sh --zookeeper ydZk:2181 --topic ydKafka --from-beginning
```

生产者

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic ydKafka
```



### RabbitMQ

```
mkdir -p /data/docker/rabbitmq/data
mkdir -p /data/docker/rabbitmq/conf
```

创建rabbitmq

```
docker run -d -p 5672:5672  -p 15672:15672 -v /data/docker/rabbitmq/data:/var/lib/rabbitmq:rw -v /etc/localtime:/etc/localtime:ro --name ydRabbitMq -e RABBITMQ_DEFAULT_USER=yd -e RABBITMQ_DEFAULT_PASS=yd@Test --restart=always rabbitmq:3.6.15-management
```

测试

```
curl http://127.0.0.1:15672
```



### Mongdb

我们先要创建一个无校验的容器，可以设置管理员。

\# 创建无校验的容器

```
docker run --name ydMongo -p 27017:27017 -v /data/docker/mongo/:/data/db -d mongo
```

```
docker exec -it ydMongo mongo admin
```

\# 创建管理员

```
db.createUser({ user:'root',pwd:'ydMongo', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });
```

\# 停止 linux-mongo 容器

```
docker stop ydMongo 
```

# 删除。其实不删除也可以，没有其他影响，不删除记得下面步骤的命名不要重复。

```
docker rm -f ydMongo
```

 

创建 MongoDB 镜像 - 带验证

\# 创建容器 - 有校验

```
docker run --name ydMongo -p 27017:27017 -v /data/docker/mongo:/data/db -d mongo --auth
```

```
docker exec -it ydMongo mongo admin
```

\# 权限认证# 返回 1 证明成功， 返回 0 证明失败

```
db.auth("root","ydMongo");
```

\# 创建可用用户。或者直接使用 rootuser ( 权限要要 root 才能在外部操作)

```
db.createUser({ user: 'testadmin', pwd: 'testadmin123', roles: [ { role: "root", db: "admin" } ] });
```

\# 校验

```
db.auth("testadmin","testadmin123");
```

 

> \# 权限说明
>
>   1.数据库用户角色：read、readWrite;
>
>   2.数据库管理角色：dbAdmin、dbOwner、userAdmin；
>
>   3.集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
>
>   4.备份恢复角色：backup、restore
>
>   5.所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
>
>   6.超级用户角色：root
>
> \# 退出
>
>   exit
>
> \# MongoDB 数据库操作
>
>   show dbs; # 查看现有数据库
>
>   show users; # 查看用户
>
>   db.dropUser("testadmin") # 删除用户，外部想连接 testadmin 的话，此处不要删除
>
> \# 至此，有一个用户可以直接使用。

 

配置外部可以连接操作服务器中的 MongoDB

\# 进入容器，注意此时不是以admin进入,也没有进入 mongodb 的shell

```
docker exec -it ydMongo bash
```

\# 更新源

```
apt-get update
```

\# 安装 vim

```
apt-get install vim
```

\# 修改 mongo 配置文件

```
vim /etc/mongod.conf.orig
```

1.确保注释掉`# bindIp: 127.0.0.1` 或者改成`bindIp: 0.0.0.0` 即可开启远程连接

2.开启权限认证

```
security：
  authorization: enabled #注意缩进，参照其他的值来改，若是缩进不对可能导致后面服务不能重启
```

\# 至此，可以通过外部操作远程 MongoDB。记得在服务器配置一下端口开放。



### hbase

```
docker run -d  --name yd_hbase -p 9181:2181  -p 16010:16010 harisekhon/hbase:1.3

docker run -d -h docker-hbase \
        -p 9181:2181 \
        -p 8080:8080 \
        -p 8085:8085 \
        -p 9090:9090 \
        -p 9000:9000 \
        -p 9095:9095 \
        -p 16000:16000 \
        -p 16010:16010 \
        -p 16201:16201 \
        -p 16301:16301 \
        -p 16020:16020\
        --name yd_hbasee \
        harisekhon/hbase:1.3
```

```
http://localhost:16010/master-status
```

```
docker exec -it yd_hbasee bash
hbase shell
```

