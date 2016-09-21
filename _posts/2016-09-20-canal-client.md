---
layout: post
date: 2016-09-20
title: Sync Mysql Data to Hbase or Kafka.
categories: [canal,hbase,kafka]
---

## Workflow
整个架构的workflow 如下

~~~ shell
[mysql master] -> [mysql slave] -> (ROW BINLOG) -> [canal server]
    -> [canal client] -> [kafka] -> [hbase/hdfs/... consumer]
~~~

Canal Server 会伪装成mysql slave，把BINLOG里面的数据解析出来，封装成CanalEntry，等待Canal Client来接。

Canal Client 负责接收Canal Server的数据，转存到kafka中。

## Q&A

#### Canal Server 挂掉，重启后是否不丢数据

#### Canal Client 挂掉，重启后是否不丢数据

#### Canal Server 端如何手动指定binlog offset

#### Canal Client 端如何手动指定binlog offset

#### 两个Canal Server 一起消费，会不会数据不全

#### HA模式下，Canal Server被kill掉后是否会丢数据

#### HA模式下，Canal Client被kill掉后是否会丢数据

## Environment

|---
|:------|:------|
| Usage| number |
| Canal Server | 2 |
| Canal Client | 2 |
| Kafka | 3 |
| ZooKeeper | 3 |
|===
| Total | 7 |

## Build

<div> Canal Server and Canal Client use different version of protobuf. You might facing the following problem </div>

~~~ shell
java.lang.UnsupportedOperationException: This is supposed to be overridden by subclasses.
        at com.google.protobuf.GeneratedMessage.getUnknownFields(GeneratedMessage.java:180) ~[protobuf-java-2.5.0.jar:na]
~~~

<div> First, modify the pom.xml </div>
~~~ xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>2.5.0</version>
</dependency>
~~~

<div> Second, generate protocol files with new protobuf, and you might need to install protocol first </div>
~~~ bash
SRC_DIR=/git/github/canal/protocol/src/main/java/com/alibaba/otter/canal/protocol
DST_DIR=/home/vagrant/protocol
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/CanalProtocol.proto
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/EntryProtocol.proto
~~~

<div>
Then, copy the generated protocol files to protocol/src/main/java/com/alibaba/otter/canal/protocol/
</div>

Finally, you can build canal. Or you can **SKIP** the steps above. Instead, you can clone [my git](https://github.com/chaopengio/canal).

~~~ bash
mvn clean install -Dmaven.test.skip -Denv=release
~~~

## Deploy

#### canal.properties Config

**common参数定义**，可以将instance.properties的公用参数，抽取放置到这里，这样每个instance启动的时候就可以共享. **instance.properties配置定义优先级高于canal.properties**

|---
| Parameter | Comment | Value |
|:---|:--------|:---|
| canal.id | 每个canal server实例的唯一标识，暂无实际意义 | 1 |
| canal.ip | canal server绑定的本地IP信息，如果不配置，默认选择一个本机IP进行启动服务 | |
| canal.port | canal server提供socket服务的端口 | 11111 |
| canal.zkServers | canal server链接zookeeper集群的链接信息 | 127.0.0.1:2181,127.0.0.1:2182 |
| canal.zookeeper.flush.period | canal持久化数据到zookeeper上的更新频率，单位毫秒 | 1000 |
| canal.file.data.dir | canal持久化数据到file上的目录，conf同目录 | ../conf |
| canal.file.flush.period | canal持久化数据到file上的更新频率，单位毫秒 | 1000 | |
| canal.instance.fallbackIntervalInSeconds | canal发生mysql切换时，mysql主备库可能存在解析延迟或者时钟不统一，需要回退一段时间，保证数据不丢 | 60 |
| canal.instance.filter.query.dcl	| 是否忽略DCL的query语句，比如grant/create user等 | false |
| canal.instance.filter.query.dml | 是否忽略DML的query语句，比如insert/update/delete table. | false |
| canal.instance.filter.query.ddl | 是否忽略DDL的query语句，比如create, drop, alter, etc | true |
| canal.instance.get.ddl.isolation | ddl语句是否隔离发送，开启隔离可保证每次只返回发送一条ddl数据，不和其他dml语句混合返回 | true |

**instance列表定义**

|---
| Parameter | Comment | Value |
|:---|:--------|:---|
| canal.destinations | 当前server上部署的instance列表 |  |
| canal.conf.dir | conf目录所在的路径 | ../conf |
| canal.auto.scan | 开启instance自动扫描如果配置为true，canal.conf.dir 目录下的instance配置变化会自动触发 | true|
| canal.auto.scan.interval | instance自动扫描的间隔时间，单位秒 | 5 |
| canal.instance.global.lazy | 全局lazy模式, 配合scan | true |


##### 重要配置介绍
- canal.auto.scan
  开启instance自动扫描，如果配置为true，canal.conf.dir目录下的instance配置变化会自动触发：
  1. instance目录新增： 触发instance配置载入，lazy为true时则自动启动
  2. instance目录删除：卸载对应instance配置，如已启动则进行关闭
  3. instance.properties文件变化：reload instance配置，如已启动自动进行重启操作

#### instance.properties Config

|---
| Parameter | Comment | Value |
|:---|:--------|:---|
| canal.instance.mysql.slaveId | 模拟mysql slave | 1234 |
| canal.instance.master.address | mysql host | 127.0.0.1:3306 |
| canal.instance.master.journal.name | mysql主库链接时起始的binlog文件 | |
| canal.instance.master.position | mysql主库链接时起始的binlog偏移量 | |
| canal.instance.master.timestamp | mysql主库链接时起始的binlog的时间戳 | |
| canal.instance.dbUsername	| | |
| canal.instance.dbPassword	| | |
| canal.instance.defaultDatabaseName | | |
| canal.instance.connectionCharset | mysql 数据解析编码 | UTF-8 |
| canal.instance.filter.regex	| mysql 数据解析关注的表，Perl正则表达式. | dbname\\.[a-zA-Z][a-zA-Z0-9_]* |

**mysql链接时的起始位置**

canal.instance.master.journal.name + canal.instance.master.position : 精确指定一个binlog位点，进行启动

canal.instance.master.timestamp : 指定一个时间戳，canal会自动遍历mysql binlog，找到对应时间戳的binlog位点后，进行启动

不指定任何信息：默认从当前数据库的位点，进行启动。(show master status)

**mysql链接的编码**

目前canal版本仅支持一个数据库只有一种编码，如果一个库存在多个编码，需要通过filter.regex配置，将其拆分为多个canal instance，为每个instance指定不同的编码
