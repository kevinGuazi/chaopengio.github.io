---
layout: post
date: 2016-09-20
title: Sync Mysql Data to Hbase or Kafka.
categories: [canal,hbase,kafka]
---

## Workflow
------
整个架构的workflow 如下

~~~ shell
[mysql master] -> [mysql slave] -> (ROW BINLOG) -> [canal server]
    -> [canal client] -> [kafka] -> [hbase/hdfs/... consumer]
~~~

Canal Server 会伪装成mysql slave，把BINLOG里面的数据解析出来，封装成CanalEntry，等待Canal Client来接。

Canal Client 负责接收Canal Server的数据，转存到kafka中。

------

## Q&A
------

#### Canal Server 挂掉，重启后是否不丢数据
**测试流程**

1. 每秒给mysql insert一条数据
2. 打开client，Server，kafka consumer
3. kill server，open server，看数据是否有丢失，或重复

**测试现象及结果**

数据没有丢，server 挂掉之后，client也挂掉了。并且重启无效，重启后提示没有alive server。

#### Canal Client 挂掉，重启后是否不丢数据
**测试流程**

1. 每秒给mysql insert一条数据
2. 打开client，Server，kafka consumer
3. kill client，open client，看数据是否有丢失，或重复

**测试现象及结果**

数据并没有丢，但是，client restart之后，有一段时间没有处理数据。怀疑是之前的挂掉之后在zk中的状态并没有改变。

**ZK观测**

client close 状态

~~~ shell
[zk: localhost:2181(CONNECTED) 6] ls /otter/canal/destinations/example/1001
[cursor]

[zk: localhost:2181(CONNECTED) 8] get /otter/canal/destinations/example/1001/cursor
{"@type":"com.alibaba.otter.canal.protocol.position.LogPosition","identity":{"slaveId":-1,"sourceAddress":{"address":"192.168.10.24","port":3306}},"postion":{"included":false,"journalName":"mysql-bin.000016","position":54624,"timestamp":1474532405000}}
~~~

这个时候 `position=54624` 然后往mysql中插入一条数据，之后，查看ZK，状态未变, `position=54624`。

说明，**_cursor_ 中 _position_ 存放的乃是当前 client 消费的位置**。

然后，打开client

~~~ shell
[zk: localhost:2181(CONNECTED) 11] ls /otter/canal/destinations/example/1001
[cursor, running]

[zk: localhost:2181(CONNECTED) 12] get /otter/canal/destinations/example/1001/running
{"active":true,"address":"192.168.10.24:35579","clientId":1001}

[zk: localhost:2181(CONNECTED) 13] get /otter/canal/destinations/example/1001/cursor
{"@type":"com.alibaba.otter.canal.protocol.position.LogPosition","identity":{"slaveId":-1,"sourceAddress":{"address":"192.168.10.24","port":3306}},"postion":{"included":false,"journalName":"mysql-bin.000016","position":54855,"timestamp":1474532678000}}
~~~

此时 _/otter/canal/destinations/example/1001_ 下多了一个running，里面记录了，被谁在消费。并且 `position=54855`, position改变了，查看client日志，确实已经被消费了。

此时，kill client，查看ZK，发现 _/otter/canal/destinations/example/1001_ 下依然还有 `running`，过了一段时间，大概一分钟后，running消失。

**结论：**

- client 的 HA 通过 ZK 下的 _/otter/canal/destinations/example/1001/running_ 来控制。Heart Beat大概为1分钟，未找到相应配置。
- client 消费的postion 同步到 _/otter/canal/destinations/example/1001/cursor_ 下。
- client 被kill掉，数据不会丢。不知道是否会重复。重复与否需要看client server更新ZK时间，默认为1s。所以可能会重复最多1s内处理的数据。

#### 两个Canal Server 一起消费，会不会数据不全
1. 创建两个server，同时读取一张表
2. 同时打开两个server
3. 分别消费两个server
4. 比较两个client的数据

Ya，no problem！

#### Canal Server 端如何手动指定binlog offset
1. 每秒给mysql insert一条数据， 过一小时后停止。记录start，end时间
2. 打开client消费所有数据
3. set `canal.instance.master.journal.name` and `canal.instance.master.timestamp`
4. stop client, stop server
5. In ZK, remove canal info. `rmr /otter`
6. start server, start client.

Everything is good！

#### Mysql 切库

------

## Environment
------

|---
|:------|:------|
| Usage| number |
| Canal Server | 2 |
| Canal Client | 2 |
| Kafka | 3 |
| ZooKeeper | 3 |
|===
| Total | 7 |

------

## Build
------

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

------

## Deploy
------

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
