# 分布式session实现之redis

参考：https://www.jianshu.com/p/5d6c7772ad87

##适用场景

- **为了使Web能适应大规模的访问,需要实现应用程序的集群部署**
- **实现集群部署首先要解决session的统一，即需要实现session的共享机制，即分布式会话**

##分布式Session的实现方式

- **基于resin/tomcat web容器本身的session复制机制**
- **基于NFS共享文件系统**
- **基于Cookie进行session共享**
- **基于数据库的Session共享**
- **基于分布式缓存的Session共享，如memcached，Redis，jbosscache**
- **基于ZooKeeper的Session共享**

<br>



