---
layout: post
date: 2016-08-31
title: NameNode Recover After Format
categories: [hadoop]
---

### 起因
我在公司hadoop集群外搭建了一个可以访问hadoop集群的机器，该机器是hadoop账号。一个新员工想在该机器上搭建测试集群，下载了新的hadoop，然而在format的时候，忘了改环境变量，直接执行了**hdf namenode -format**。然后世界就崩溃了。集群的namenode被format了。

## 解决
经过多方调研，最后解决方案是：按照journalnode中的数据更改
- namenode中/data/hadoop/dfs/name/current/VERSION 的namespaceID，clusterID，
- 更改datanode中/data*/hadoop/dfs/current/BP-1378121585-10.1.1.12-1464255691021/current/VERSION，和/data*/hadoop/dfs/current/VERSION 中的namespaceID和clusterID。

### 经过
当时我正在面试，忽闻spark job连接不上hdfs。我转头查看namenode nn1已经挂掉。事情比较急，我直接尝试启动namenode nn1，发现启动不了。也没有去看为何nn1会挂掉，转头准备把namenode nn2切换为active状态，然而造成nn2也挂掉。回头想想，如果先查看日志发现被format后，可能从nn2 copy metadata 到nn1就能解决问题了，当然这个也没验证过，存疑。检查方法：看nn2 中VERSION状态。

扯远了，回到正题。nn2挂掉之后，我知道事情不简单了，回头看nn1的日志，发现nn1 的namenode log中出现：

```shell
2016-08-31 15:17:44,930 INFO org.apache.hadoop.hdfs.server.namenode.FSNamesystem: Roll Edit Log from 10.1.1.13
2016-08-31 15:17:44,930 INFO org.apache.hadoop.hdfs.server.namenode.FSEditLog: Rolling edit logs
2016-08-31 15:17:44,930 INFO org.apache.hadoop.hdfs.server.namenode.FSEditLog: Ending log segment 36230724
2016-08-31 15:17:44,930 INFO org.apache.hadoop.hdfs.server.namenode.FSEditLog: Number of transactions: 61 Total time for transactions(ms): 4 Number of transactions batched in Syncs: 0 Number of syncs: 24 SyncTimes(ms): 323 227
2016-08-31 15:17:44,945 INFO org.apache.hadoop.hdfs.server.namenode.FSEditLog: Number of transactions: 61 Total time for transactions(ms): 4 Number of transactions batched in Syncs: 0 Number of syncs: 25 SyncTimes(ms): 333 233
2016-08-31 15:17:44,951 INFO org.apache.hadoop.hdfs.server.namenode.FileJournalManager: Finalizing edits file /data/hadoop/dfs/name/current/edits_inprogress_0000000000036230724 -> /data/hadoop/dfs/name/current/edits_0000000000036230724-0000000000036230784
2016-08-31 15:17:44,951 INFO org.apache.hadoop.hdfs.server.namenode.FSEditLog: Starting log segment at 36230785
2016-08-31 15:19:36,764 INFO org.apache.hadoop.hdfs.server.namenode.FSEditLog: Number of transactions: 2 Total time for transactions(ms): 1 Number of transactions batched in Syncs: 0 Number of syncs: 1 SyncTimes(ms): 7 42
2016-08-31 15:19:36,816 WARN org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager: Remote journal 10.1.1.12:8485 failed to write txns 36230786-36230786. Will try to write to this JN again after the next log roll.
org.apache.hadoop.ipc.RemoteException(java.io.IOException): IPC's epoch 1 is not the current writer epoch  0
    at org.apache.hadoop.hdfs.qjournal.server.Journal.checkWriteRequest(Journal.java:449)
    at org.apache.hadoop.hdfs.qjournal.server.Journal.journal(Journal.java:341)
    at org.apache.hadoop.hdfs.qjournal.server.JournalNodeRpcServer.journal(JournalNodeRpcServer.java:148)
    at org.apache.hadoop.hdfs.qjournal.protocolPB.QJournalProtocolServerSideTranslatorPB.journal(QJournalProtocolServerSideTranslatorPB.java:158)
    at org.apache.hadoop.hdfs.qjournal.protocol.QJournalProtocolProtos$QJournalProtocolService$2.callBlockingMethod(QJournalProtocolProtos.java:25421)
    at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:616)
    at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:969)
    at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2049)
    at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2045)
    ...
    ...
2016-08-31 15:19:36,876 FATAL org.apache.hadoop.hdfs.server.namenode.FSEditLog: Error: flush failed for required journal (JournalAndStream(mgr=QJM to [10.1.1.12:8485, 10.1.1.13:8485, 10.1.1.14:8485], stream=QuorumOutputStream starting at txid 36230785))
...
...
2016-08-31 15:19:36,876 WARN org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager: Aborting QuorumOutputStream starting at txid 36230785
2016-08-31 15:19:36,906 INFO org.apache.hadoop.util.ExitUtil: Exiting with status 1
2016-08-31 15:19:36,989 INFO org.apache.hadoop.hdfs.server.namenode.NameNode: SHUTDOWN_MSG:
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at hadoop-01.hadoop-01/10.1.1.12
************************************************************/
2016-08-31 15:23:45,695 INFO org.apache.hadoop.hdfs.server.namenode.NameNode: STARTUP_MSG:
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = hadoop-01/10.1.1.12
STARTUP_MSG:   args = []
STARTUP_MSG:   version = 2.7.2
```

而后有

```shell
2016-08-31 15:23:54,199 INFO org.apache.hadoop.hdfs.server.blockmanagement.DatanodeDescriptor: Adding new storage ID DS-9997904d-4414-4153-ad87-cd8d447d6036 for DN 10.1.1.12:50010
2016-08-31 15:23:54,199 INFO org.apache.hadoop.hdfs.server.blockmanagement.DatanodeDescriptor: Adding new storage ID DS-0ba7e77f-862f-42f9-9b8b-924c78b6d8e3 for DN 10.1.1.12:50010
2016-08-31 15:23:54,199 INFO org.apache.hadoop.hdfs.server.blockmanagement.DatanodeDescriptor: Adding new storage ID DS-523694de-3a9f-480f-833b-9c225380154f for DN 10.1.1.12:50010
```

而后有

```shell
2016-08-31 15:31:28,819 INFO org.apache.hadoop.ipc.Server: IPC Server handler 5 on 8020, call org.apache.hadoop.hdfs.protocol.ClientProtocol.renewLease from 10.1.1.15:9306 Call#15914 Retry#0: org.apache.hadoop.ipc.StandbyException: Operation category WRITE is not supported in state standby
2016-08-31 15:31:29,216 INFO org.apache.hadoop.ipc.Server: IPC Server handler 3 on 8020, call org.apache.hadoop.hdfs.protocol.ClientProtocol.getListing from 10.1.1.14:27624 Call#38523 Retry#14: org.apache.hadoop.ipc.StandbyException: Operation category READ is not supported in state standby
```

由于当时还在面试，并没有仔细分析这些log，回头想想真是万幸。

而后才有

```shell
2016-08-31 15:31:29,345 INFO org.apache.hadoop.hdfs.server.namenode.FSNamesystem: Starting services required for active state
2016-08-31 15:31:29,362 INFO org.apache.hadoop.hdfs.qjournal.client.QuorumJournalManager: Starting recovery process for unclosed journal segments...
2016-08-31 15:31:29,381 FATAL org.apache.hadoop.hdfs.server.namenode.FSEditLog: Error: recoverUnfinalizedSegments failed for required journal (JournalAndStream(mgr=QJM to [10.1.1.12:8485, 10.1.1.13:8485, 10.1.1.14:8485], stream=null))
org.apache.hadoop.hdfs.qjournal.client.QuorumException: Got too many exceptions to achieve quorum size 2/3. 3 exceptions thrown:
10.1.1.12:8485: Incompatible namespaceID for journal Storage Directory /data/hadoop/qjm/data/ns1: NameNode has nsId 1758148174 but storage has nsId 1633807959
```

这个时候我才知道，namenode可能被format了。万分庆幸在重启namenode之后没有打开active，和safemode。不然数据就全没了。


仔细查了一下journal node的数据，namenode 的metadata，datanode的数据

```shell
hdfs@hadoop-01:~$ cat /data/hadoop/qjm/data/ns1/current/VERSION
#Wed Aug 31 15:17:45 CST 2016
namespaceID=1633807959
clusterID=CID-161b976f-c040-4bf4-80fc-194256533d01
cTime=0
storageType=JOURNAL_NODE
layoutVersion=-63

hdfs@hadoop-01:~$ cat /data/hadoop/dfs/name/current/VERSION
#Wed Aug 31 14:57:36 CST 2016
namespaceID=1758148174
clusterID=CID-884a8bbc-527e-467c-8a5d-52138a3a3c55
cTime=0
storageType=NAME_NODE
blockpoolID=BP-1378121585-10.1.1.12-1464255691021
layoutVersion=-63

hdfs@hadoop-04:~$ cat /data1/hadoop/dfs/current/BP-1378121585-10.1.1.12-1464255691021/current/VERSION
#Wed Aug 31 14:57:36 CST 2016
namespaceID=1758148174
cTime=0
blockpoolID=BP-1378121585-10.1.1.12-1464255691021
layoutVersion=-56

hdfs@hadoop-04:~$ cat /data1/hadoop/dfs/current/VERSION
#Wed Aug 31 19:32:22 CST 2016
storageID=DS-4b37d7f4-09a0-446f-a453-97c1d86449b3
clusterID=CID-884a8bbc-527e-467c-8a5d-52138a3a3c55
cTime=0
datanodeUuid=95efdf57-5008-456b-b6ed-52079cd5b0e1
storageType=DATA_NODE
layoutVersion=-56
```

发现一切都乱了。查了很多资料，发现有很多种解决方案。

1. hadoop namenode -recover。先backup fsimage，然后执行recover。
2. hdfs namenode -importCheckpoint。从fsimage中恢复。
3. 更改namenode中/data/hadoop/dfs/name/current/VERSION 的namespaceID，clusterID，清空journalnode中的数据，然后重启namenode。
4. 按照journalnode中的数据更改namenode中/data/hadoop/dfs/name/current/VERSION 的namespaceID，clusterID，更改datanode中/data1/hadoop/dfs/current/BP-1378121585-10.1.1.12-1464255691021/current/VERSION，和/data1/hadoop/dfs/current/VERSION 中的namespaceID和clusterID。

1,2中，我试了方案 1。
在做 1 的时候，我把namenode中的namespaceID和clusterID改到了最原始版本，可是没有添加blockPoolID，当时并不知道blockPoolID可以在datanode中找到。于是，就没有操作成功，现在想来应该是可以的。不过已经没有办法证实了。

然后在选择3，4中，我选择了4。没有太多的比较，只是觉得4工作量虽大，但是不涉及到journalnode，可能会更好些，而且也期待数据不会丢失。
于是，在每台机修改了24个文件之后，终于成功的启动了。

## 反思
- 在production 环境中，一定要配置dfs.namenode.support.allow.format 为false。
- 权限管理非常重要。（这一点还没想好如何做）
- namenode 挂掉一定要第一时间查找原因，不确定前千万不能离开safemode，不能设置active，第一时间查看namenode VERSION。
- 做测试一定要新建虚拟机，在全新虚拟机上做测试，极大可能有自己不知道的配置。copy配置过去之后，第一时间吧相应配置replace掉。
