---
layout: post
date: 2016-08-27
title: 在cloudera上看到的一篇讲如何优化运行spark的文章
categories: [spark, yarn]
---
## Tuning Spark

Spark 关注两种资源：CPU，Memory。磁盘和网络当然也很重要，不过spark、yarn在这上面没做任何事情。
一个应用中的每一个spark executor 都有相同个数的 core 和相同的 heap size。

- **core** 的个数由 **\-\-executor-cores** 来指定，否则将用 **spark-defaults.conf** 中的 **spark.executor.cores**。
- **heap size** 由 **\-\-executor-memory**, 或 **spark.executor.memory**。

Define：

- **\-\-executor-cores** 一个 executer 最多可以同时运行多少tasks。
- **\-\-executoer-memory** spark 最多能 cache 的 size，最多能用于 shuffle 的内存.
- **\-\-num-executors** 或 **spark.executor.instances** 一共需要多少 executor。 或者指定dynamicAllocation，帮助我们自行调整executors。

在优化使用资源之前，首先要了解我们一共有多少资源可用。这些都在yarn-site.xml中

 - **yarn.nodemanager.resource.memory-mb** 每台机最多可被 container 使用的内存
 - **yarn.nodemanager.resource.cpu-vcores** 每台机最多可被 container 使用的内核

此外，yarn 还需要一部分memory来做其他事情，比如字符串常量池和 direct byte buffers等。spark executor 为yarn准备的这部分内存名叫spark.yarn.executor.memoryOverhead，并有如下计算公式。

- spark.yarn.executor.memoryOverhead = max（384, spark.executor.memory * 0.07）

对 yarn 资源和 spark 资源，有如下公式。

 - yarn.nodemanager.resource.memory-mb = spark.yarn.executor.memoryOverhead + spark.executor.memory
 - spark.executor.memory = spark.shuffle.memoryFraction + spark.storage.memoryFraction

此外，还有些因素要考虑。

 - **application master** 这是一个 non-executor 的 container，需要独立申请资源。yarn-client 模式下，默认 1G 内存、1个核。yarn-cluster 模式下，需要用 **\-\-driver-memory** 和 **\-\-driver-cores** 来设定。
 - executor 给的内存太多会造成大量的GC延迟，大概估计一个executor不要超过64G。
 - HDFS client 在大量多线程下会有性能问题。大概估计每个executor不能超过5个tasks。
 - 跑大量小的 executor（只有一个task）比跑多个task的executor 性能差。因为这样会让很多broadcast的数据在内存中存储多次。

### Example：6台机，每台机64G，16 cores。

#### 计算方法：
每台机留1个core，1G 作为其他使用。
每个task 1个core，那么一台机最多15个task。一个executor 5个tasks，所以，一台机 3个executor最多。
留出一个executor的空间作为application master。

 - \-\-num-executors = 6 * (15/5) - 1 = 17。

一台机总共63G。每个executor 最多可用63/3 = 21G，根据公式

 - yarn.nodemanager.resource.memory-mb = spark.yarn.executor.memoryOverhead + spark.executor.memory
 - \-\-executor-memory = 21 / 1.0.7 = 19G。



#### Ref:

- [how-to-tune-your-apache-spark-jobs-part-2](http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-2/comment-page-1/#comment-73052)
