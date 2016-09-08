---
layout: post
date: 2016-09-06
title: 跑spark job的时候，如何配置自动分配executor。
categories: [spark, yarn]
---
## Application Config
- Set **_spark.dynamicAllocation.enabled_** to **_true_**
- Set **_spark.shuffle.service.enabled_** to **_true_**

## Server Config
### Set up an **_external shuffle service_** on each worker node.

这是为了在删除shuffle executor之后，不删除其写的数据。
1. 确保编译spark的时候加入了-Dyarn参数。
2. 找到spark-<version>-yarn-shuffle.jar，应该在$SPARK_HOME/lib下面
3. 把该jar包放到nodemanagers的classpath下。在 _yarn-site.xml_ 中 _yarn.application.classpath_ 下添加 _$YARN_HOME/share/hadoop/spark/*_，然后把jar拷到相应目录下。
4. 在 _yarn-site.xml_ 中，添加 _spark_shuffle_ 到 _yarn.nodemanager.aux-services_ 中，并设置 _yarn.nodemanager.aux-services.spark_shuffle.class_ 为 _org.apache.spark.network.yarn.YarnShuffleService_
5. Restart nodemanager.

## Reference
- [https://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation](https://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation)
