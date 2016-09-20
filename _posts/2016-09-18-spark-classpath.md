---
layout: post
date: 2016-09-18
title: Spark ClassPath 加载顺序
categories: [spark]
---

## 背景
前段时间写了点spark的job，需要跑在yarn上，然而遇到了一些guava依赖的问题。然后回忆起了之前听 **田毅** 讲的一些spark classpath的问题。翻了一下历史记录，记了下来。

## spark-submit 提交的任务的classpath
1. $SPARK_HOME/lib/datanucleus.\*.jar
2. $SPARK_CLASSPATH (deprecated)
3. --driver-classpath
4. --jars
5. spark.executor.extraClassPath
6. spark.driver.extraClassPath

## spark-submit 执行顺序
1. bin/spark-submit
2. bin/spark-class
3. bin/load-spark-env.sh
4. conf/spark-env.sh
5. java -cp ... org.apache.spark.launcher.main
6. 生成Driver端的启动命令

## Driver的classpath加载顺序
1. $SPARK_CLASSPATH (deprecated)
2. --driver-classpath or spark.driver.extraClassPath
3. $SPARK_HOME/conf
4. $SPARK_HOME/lib/spark-assembly-xxx-hadoopxxx.jar
5. $SPARK_HOME/lib/datanucleus.\*.jar
6. $HADOOP_CONF_DIR
7. $YARN_CONF_DIR
8. $SPARK_DIST_CLASSPATH

## Executor的classpath加载顺序
Executor的加载远比Driver的要复杂。
1. spark.executor.extraClassPath
2. $SPARK_HOME/lib/spark-assembly-xxx-hadoopxxx.jar
3. $HADOOP_CONF_DIR
4. hadoop classpath
5. --jars

## 如何控制jar包
 - **Driver**: 用 **--driver-class-path** 来控制driver端的classpath
 - **Executor**: 如果用 **--jars**，需要注意和hadoop中和spark-assembly的类冲突问题，如果需要优先加载，通过 **spark.executor.extraClassPath** 方式进行配置。

## Guava 依赖

|-----+-----|
| Package | Guava Version |
|:---------:|:--------------:|
|Hadoop-2.7.2| 11.0.2|
|Hbase-1.1.5| 12.0.1|
|Spark-core_2.11-1.6.2|14.0.1|
|-----+-----|

每个package所依赖的guava版本不同，还好，我的spark job中不曾遇到更高版本的guava。所以该job的pom.xml中exclue所有guava包，并且在spark-submit中加上 **--driver-class-path /path/to/guava-14.0.jar**。

有些package，比如elasticsearch需要用到guava-18.0，这个版本的guava和老的版本是无法兼容的，老版本有些class，有些method，没有在新版本中用到。这个时候就需要去修改package的源代码，重新打包。
