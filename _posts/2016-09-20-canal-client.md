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
| Parameter | Comment | Default | Note |
|:---|:--------|:---|:--------|
| canal.id | 每个canal server实例的唯一标识，暂无实际意义 | 1 | HA 下要改ID |
| canal.ip | canal server绑定的本地IP信息，如果不配置，默认选择一个本机IP进行启动服务 | x | 设置为本机ip |
| canal.port | canal server提供socket服务的端口 | 11111 | |
| canal.zkServers | canal server链接zookeeper集群的链接信息
例子：127.0.0.1:2181,127.0.0.1:2182 | x | |
| canal.zookeeper.flush.period | canal持久化数据到zookeeper上的更新频率，单位毫秒 | 1000 | 如果数据量较大，可以适当调小 |
| canal.file.data.dir | canal持久化数据到file上的目录 | ../conf (默认和instance.properties为同一目录，方便运维和备份) | 不知道和zk会不会重复，是否一个就够了 |
| canal.file.flush.period | canal持久化数据到file上的更新频率，单位毫秒 | 1000 | |
| canal.instance.fallbackIntervalInSeconds | canal发生mysql切换时，在新的mysql库上查找binlog时需要往前查找的时间，单位秒
说明：mysql主备库可能存在解析延迟或者时钟不统一，需要回退一段时间，保证数据不丢 | 60 | |
| canal.instance.filter.query.dcl	| 是否忽略DCL的query语句，比如grant/create user等 | false | 如果不需要监控user信息，建议打开 |
| canal.instance.filter.query.dml | 是否忽略DML的query语句，比如insert/update/delete table.(mysql5.6的ROW模式可以包含statement模式的query记录)| false | 数据filter，不要打开 |
| canal.instance.filter.query.ddl | 是否忽略DDL的query语句，比如create table/alater table/drop table/rename table/create index/drop index. (目前支持的ddl类型主要为table级别的操作，create databases/trigger/procedure暂时划分为dcl类型) | false | 我司用了一个恶心的工具，在修改表结构的时候会产生一些以下划线开头的表作为临时表，这些表会让canal server崩溃。所以我打开了 |
| canal.instance.get.ddl.isolation | ddl语句是否隔离发送，开启隔离可保证每次只返回发送一条ddl数据，不和其他dml语句混合返回.(otter ddl同步使用) | false | 设为true吧 |
