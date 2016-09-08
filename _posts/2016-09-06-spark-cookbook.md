---
layout: post
date: 2016-09-06
title: 写spark时遇到的各种注意事项。
categories: [spark, sql]
---
## spark sql中的tasks个数
spark 在执行sql时，默认shuffle个数为200，如果数据量没这么大，资源没这么多，比较耗时。如果你有5个sql要做，那么task的个数会变成1000个。设置shuffle个数的方法如下。

```python
sqlContext.sql("set spark.sql.shuffle.partitions=10");
```

## spark在同一个table中执行多个sql时，table是否会重新计算？
会！

## repartition vs coalesce
- **repartition**: can do both increase and decrease
- **coalesce**: can only do decrease
- **coalesce**: can void **full** shuffle

## Spark sql 执行 **left join** 之后，总有一个task分的数量非常多，数据偏移非常严重。
分析后发现是由于，join key是int型，默认shuffle的算法对int支持不好。换成string之后问题解决。

## spark join 之后，java做repartition方法
```java
// java
table.repartition(table.col(partitionKey));
```
