# Solr vs. Elasticsearch —>谁是开源搜索引擎王者

<br><br>

## Apache Solr

Apache Solr基于业界大名鼎鼎的java开源搜索引擎Lucene，Lucene更多的是一个软件包，还不能称之为搜索引擎，而solr则完成对lucene的封装，是一个真正意义上的搜索引擎框架。在过去的十年里，solr发展壮大，拥有广泛的用户群体。solr提供分布式索引、分片、副本集、负载均衡和自动故障转移和恢复功能。如果正确部署，良好管理，solr就能够成为一个高可靠、可扩展和高容错的搜索引擎。不少互联网巨头，如Netflix，eBay，Instagram和Amazon（CloudSearch）均使用Solr。

solr的主要特点：

- 全文索引
- 高亮
- 分面搜索
- 实时索引
- 动态聚类
- 数据库集成
- NoSQL特性和丰富的文档处理（例如Word和PDF文件）

<br>

## Elasticsearch

与solr一样，Elasticsearch构建在Apache Lucene库之上，同是开源搜索引擎。Elasticsearch在Solr推出几年后才面世的，通过REST和schema-free（**不需要预先定义 Schema，solr是需要预先定义的**）的JSON文档提供分布式、多租户全文搜索引擎。并且官方提供Java，Groovy，PHP，Ruby，Perl，Python，.NET和Javascript客户端。

分布式搜索引擎包含可以华为为分片（shard）的索引，每一个分片可以有多个副本（replicas）。每个Elasticsearch节点可以有一个或多个分片，其引擎既同时作为协调器（coordinator ），将操作转发给正确的分片。

Elasticsearch可扩展为准实时搜索引擎。其中一个关键特性是多租户功能，可根据不同的用途分索引，可以同时操作多个索引。

Elasticsearch主要特性：

- 分布式搜索
- 多租户
- 查询统计分析
- 分组和聚合

<br>

## 热度对比

在开始比较前，我们可以查看两者在google中的[搜索热度](https://trends.google.com/trends/explore?date=all&q=apache%20solr,elasticsearch)，可以看出在2013年后，Elasticsearch与Solr相比具有很大的吸引力，但这并不意味着Apache Solr已经死了。虽然不少人不认可，但Solr仍然是最流行的搜索引擎之一，具有强大的开源社区支持。

<br>

##安装与配置

相对来说，Elasticsearch更易于安装，与Solr相比非常轻量级。 Solr的分发软件包大小的当前版本（[6.4.2](http://mirrors.hust.edu.cn/apache/lucene/solr/6.4.2/)）大约为150 MB，而Elasticsearch分发软件包大小的当前版本（[5.2.2](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.2.2.tar.gz)）仅为32.2MB。

但是，如果Elasticsearch管理不好，这种易于部署和使用可能会成为一个问题。基于JSON的配置很容易，但如果你想为文件中的每个配置指定注释，那么它不适合你。Solr也提供了Rest API，可以通过集合API创建自定义分片集合，记录聚类算法和执行自定义分片。

总的来说，如果你的应用程序使用JSON，那么Elasticsearch是一个更好的选择。否则，使用Solr，因为它的schema.xml和solrconfig.xml有很好的文档。

<br>

##索引和搜索

###数据源

Solr接受来自不同来源的数据，包括XML文件，逗号分隔符（CSV）文件和从数据库中的表提取的数据以及常见的文件格式（如Microsoft Word和PDF）。

Elasticsearch还支持其他来源的数据，例如ActiveMQ，AWS SQS，DynamoDB（Amazon NoSQL），FileSystem，Git，JDBC，JMS，Kafka，LDAP，MongoDB，neo4j，RabbitMQ，Redis，Solr和Twitter。还有各种插件可用。

###搜索

**Solr专注于文本搜索，而Elasticsearch则常用于查询、过滤和分组** **分析统计**，Elasticsearch背后的团队也努力让这些查询更为高效。因此当比较两者时，对那些不仅需要文本搜索，同时还需要复杂的时间序列搜索和聚合的应用程序而言，毫无疑问Elasticsearch是最佳选择。

###索引

两者都支持使用停用词和同义词来匹配文档。

在Solr中，索引间进行join必须是单个分片和其他节点上的副本集进行关联来搜索文档间关系（例如SQL连接）。而Elasticsearch提供更高效的has_children和top_children查询来检索这样的相关文档。

<br>

##可扩展性和分布式

搜索引擎需要处理数以百万级的文档，基于此搜索引擎应该是可复制的，模块化的和可扩展的，支持集群和分布式架构。

###专为云而设计

Elasticsearch非常易于扩展，拥有足够多的需要大集群的使用案例。

Solr 基于Apache ZooKeeper也实现了类似ES的分布式部署模式。ZooKeeper是成熟和广泛使用的独立应用程序。

相对比，Elasticsearch有一个内置的类似ZooKeeper的名为Zen的组件，通过内部的协调机制来维护集群状态。

可以说Elasticsearch是转为云而设计，是分布式首选。

###分片拆分和再平衡

shards是luence索引的分区单元，solr和elasticsearch均使用。你可以通过在集群中的不同计算机上运行shard来分发索引。随着SolrCloud的引入，Solr开始支持shard拆分，这允许您通过拆分现有shard来添加更多shard。相比之下，ElasticSearch仍然不支持这一点，事实上，实际上阻止了这种做法。ES通过向设置中添加更多计算机，可以使用自动碎片平衡功能。相比之下，Solr允许添加分片（使用隐式路由时）或分割（使用复合ID时），但不能删除分片。它允许您增加副本。在Elasticsearch中，默认情况下每个索引具有五个分片。它不允许您更改主分片的数量，但它允许您增加副本的数量。分片再平衡对于水平扩容非常有用。当添加新机器时，它将自动重新平衡不同机器中可用的分片。

<br>

##社区

Solr有一个广泛的开源社区。任何人都可以贡献给Solr，新的Solr开发人员或代码提交者只能根据功能选择。 Elasticsearch在技术上是开源的，但不完全。所有贡献者都可以访问源代码，用户可以进行更改并提供。但最终的变化由Elastic（运行Elasticsearch和其他软件的公司）的员工确认和完成。因此，Elasticsearch更多地由单个公司驱动，而不是整个社区。

Solr贡献者和提交者跨越多个组织，而Elasticsearch提交者仅来自Elastic。还有人指出，Solr的强大社区有一个健康的项目管道和许多知名公司参与。这些成员还通过在整个开发和工程过程中做出贡献来投资该平台。

两者都有很好的用户群和丰富的开发人员社区，但ElasticSearch相较于Solr更新。 Solr已经存在了更长的时间，所以它的生态系统是发达的，拥有更大的用户群。

<br>

## 总结

- Solr 是传统搜索应用的有力解决方案，但 Elasticsearch 更适用于新兴的实时搜索应用。
- Solr 利用 Zookeeper 进行分布式管理，而 Elasticsearch 自身带有分布式协调管理功能;
- Solr 支持更多格式的数据，而 Elasticsearch 仅支持json文件格式；
- Solr 官方提供的功能更多，而 Elasticsearch 本身更注重于核心功能，高级功能多有第三方插件提供；
- Solr 在传统的搜索应用中表现好于 Elasticsearch，但在处理实时搜索应用时效率明显低于 Elasticsearch。
- Solr有一个更大、更成熟的用户、开发和贡献者社区。比较成熟、稳定。
- Solr支持添加多种格式的索引，如：HTML、PDF、微软 Office 系列软件格式以及 JSON、XML、CSV 等纯文本格式。
- Solr如果不考虑建索引的同时进行搜索，速度更快。

<br>

## 选Solr 还是 Elasticsearch?

其实，无论选择Solr还是Elasticsearch，你首先需要了解您的用户场景和未来的需求。

请记住：

- Elasticsearch由于其易用性而在较新的开发人员中更受欢迎
- 但是如果你已经在使用solr了，请继续使用它，因为迁移到Elasticsearch并不会带来具体的优势
- 如果您需要它来处理分析查询以及搜索文本，Elasticsearch是更好的选择，特别是收集日志，做分析处理

总之，两者都是功能丰富的搜索引擎，并且或多或少地给出相同的性能，只要它们被设计和实施得很好。 

<br>





