---
layout: post
date: 2016-09-19
title: Spark Distributed Cache
categories: [spark]
---

## Background
spark job每次提交的时候，会需要把 **--jars** 包从上传到hdfs上，然后在执行时拉到每台机器上。对比较常用的jar包来说，这些操作是浪费的。

## Distributed Cache
spark 1.6中提供了很多配置项，在**_spark-defaults.conf_**中
- **spark.yarn.jar**：spark jar文件存放在hdfs中的地址。避免每次启动都要从local copy过去。
- **spark.yarn.dist.files**，**spark.yarn.dist.archives**：会把文件放到每个executor中。对jar包无效。
- **spark.driver.extraClassPath**, **spark.executor.extraClassPath**: 常用jar包的主要配置方式，在**_spark-defaults.conf_**中配置，通过 **:** 隔开，其中为local地址，在每台机上放置。每个executor，driver会从local的相应位置去获取jar包地址。可以存放一些通用的jar包，比如hbase，mysql，datanucleus，guava，kafka，lzo等包。

## Run Job
在跑spark job的时候要添加 **hive-site.xml**, 且注意要同时设置 **spark.executor.extraClassPath** 和 **spark.driver.extraClassPath**。

## Example
```shell
spark-submit \
    --master yarn \
    --deploy-mode cluster \
    --name yarn-test \
    --executor-cores 3 \
    --executor-memory 6g \
    --num-executors 5 \
    --driver-memory 6g \
    --files $SPARK_HOME/conf/hive-site.xml \
    --class com.guazi.cataract.Cataract \
    cataract-0.0.1-SNAPSHOT-jar-with-dependencies.jar
```

## Problems & Solutions
> spark.executor.extraClassPath ClassLoaderResolver for class "" gave error on creation : {1}

可能是由于没有添加hive-site.xml，或者没有设置spark.driver.extraClassPath

通过 ‘**:**’ 隔开extraClassPath。
