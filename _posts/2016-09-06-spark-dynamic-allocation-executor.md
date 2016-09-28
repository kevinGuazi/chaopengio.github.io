---
layout: post
date: 2016-09-06
title: 跑spark job的时候，如何配置自动分配executor。
categories: [spark, yarn]
---
## Application Config
- Set **_spark.dynamicAllocation.enabled_** to **_true_**
- Set **_spark.shuffle.service.enabled_** to **_true_**

```shell
spark-submit \
    --master yarn \
    --deploy-mode cluster \
    --name yarn-test \
    --conf spark.dynamicAllocation.enabled=true \
    --conf spark.shuffle.service.enabled=true \
    --executor-cores 5\
    --executor-memory 14g \
    --driver-memory 6g \
    --jars ~/guava-14.0.jar,$SPARK_HOME/lib/datanucleus-api-jdo-3.2.6.jar,$SPARK_HOME/lib/datanucleus-rdbms-3.2.9.jar,$SPARK_HOME/lib/datanucleus-core-3.2.10.jar \
    --class com.guazi.cataract.Cataract cataract-0.0.1-SNAPSHOT-jar-with-dependencies.jar
```

## Server Config

Set up an **_external shuffle service_** on each worker node.

这是为了在删除shuffle executor之后，不删除其写的数据。

1. 确保编译spark的时候加入了-Dyarn参数。
2. 找到spark-<version>-yarn-shuffle.jar，应该在$SPARK_HOME/lib下面
3. 把该jar包放到nodemanagers的classpath下。然后把jar拷到 **_$YARN_HOME/share/hadoop/yarn/_** 目录下。
4. 在 _yarn-site.xml_ 中，添加 _spark_shuffle_ 到 _yarn.nodemanager.aux-services_ 中，并设置 _yarn.nodemanager.aux-services.spark_shuffle.class_ 为 _org.apache.spark.network.yarn.YarnShuffleService_
5. Restart nodemanager.

## Reference
- [https://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation](https://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation)

## It DID NOT work
最开始一切都还好，提交了一个job，能够自己根据集群资源分配到资源。可是，一旦有需要kill 掉这个application之后，所有nodemanager都报错了。

## Resource Allocation Policy
At a high level, Spark should relinquish executors when they are no longer used and acquire executors when they are needed. Since there is no definitive way to predict whether an executor that is about to be removed will run a task in the near future, or whether a new executor that is about to be added will actually be idle, we need a set of heuristics to determine when to remove and request executors.

在上层，spark控制着executors，长时间不用时移除，需要时申请。由于，我们并没有确定的方法去预测什么时候一个即将被移除的executor会很快被使用，或者预测一个即将加进来的executor实际上会被闲置，因此，我们需要一些启发式的算法来决定什么时候移除和申请executors。

#### Request Policy 申请策略
A Spark application with dynamic allocation enabled requests additional executors when it has pending tasks waiting to be scheduled. This condition necessarily implies that the existing set of executors is insufficient to simultaneously saturate all tasks that have been submitted but not yet finished.

设置为动态分配策略的spark application 允许申请额外的executors 当其有pending tasks的时候。这种情况暗指已经有的executors不够同时处理所有已提交的未完成的tasks。

Spark requests executors in rounds. The actual request is triggered when there have been pending tasks for `spark.dynamicAllocation.schedulerBacklogTimeout` seconds, and then triggered again every `spark.dynamicAllocation.sustainedSchedulerBacklogTimeout` seconds thereafter if the queue of pending tasks persists. Additionally, the number of executors requested in each round increases exponentially from the previous round. For instance, an application will add 1 executor in the first round, and then 2, 4, 8 and so on executors in the subsequent rounds.

Spark 是一轮一轮的申请executors的。当有tasks pending了`spark.dynamicAllocation.schedulerBacklogTimeout`秒之后，会触发一轮申请，如果还有pending tasks，过`spark.dynamicAllocation.sustainedSchedulerBacklogTimeout`秒之后，会再次触发一轮申请。每轮申请都是指数式上涨的。比如，一个application会在第一轮添加1个，然后在之后每轮分别添加2，4，8...

The motivation for an exponential increase policy is twofold. First, an application should request executors cautiously in the beginning in case it turns out that only a few additional executors is sufficient. This echoes the justification for TCP slow start. Second, the application should be able to ramp up its resource usage in a timely manner in case it turns out that many executors are actually needed.

一个指数增加策略的动机是双重的。第一，一个application应该在最开始的时候小心的申请资源，以防很多无用的资源被申请。其原因与TCP启动慢一样。第二，application要逐渐增大资源申请，以防真的需要很多资源。

#### Remove Policy 移除策略
The policy for removing executors is much simpler. A Spark application removes an executor when it has been idle for more than `spark.dynamicAllocation.executorIdleTimeout` seconds. Note that, under most circumstances, this condition is mutually exclusive with the request condition, in that an executor should not be idle if there are still pending tasks to be scheduled.

移除策略要简单得多。当一个executor空闲时间超过`spark.dynamicAllocation.executorIdleTimeout`秒之后，就会被移除。注意，在大多数情况下，这种条件和申请资源的条件不会是同时存在的。如果有pending task，executor不会空闲。

#### Graceful Decommission of Executors
Before dynamic allocation, a Spark executor exits either on failure or when the associated application has also exited. In both scenarios, all state associated with the executor is no longer needed and can be safely discarded. With dynamic allocation, however, the application is still running when an executor is explicitly removed. If the application attempts to access state stored in or written by the executor, it will have to perform a recompute the state. Thus, Spark needs a mechanism to decommission an executor gracefully by preserving its state before removing it.

在动态分配之前，spark executor会一直存在，不管executor出错了还是相应application退出了。但是在两种情况下，executor的全部状态都不必存在，可以被安全的移除。在动态分配下，executor被移除后，application可以继续跑。如果application尝试去获取executor存下来的状态，它会重新计算这个状态。因此，spark需要一个机制来移除executor但保留其state。

This requirement is especially important for shuffles. During a shuffle, the Spark executor first writes its own map outputs locally to disk, and then acts as the server for those files when other executors attempt to fetch them. In the event of stragglers, which are tasks that run for much longer than their peers, dynamic allocation may remove an executor before the shuffle completes, in which case the shuffle files written by that executor must be recomputed unnecessarily.

这个需求对shuffle尤其重要。在shuffle中，executor首先把自己的map结果写到本地磁盘中，然后充当一个服务器，给其他executor提供数据。由于总有些慢的tasks，dynamic allocation可能已经移除了那些先跑完的executor，那么这些executor可能需要重新被执行。

The solution for preserving shuffle files is to use an external shuffle service, also introduced in Spark 1.2. This service refers to a long-running process that runs on each node of your cluster independently of your Spark applications and their executors. If the service is enabled, Spark executors will fetch shuffle files from the service instead of from each other. This means any shuffle state written by an executor may continue to be served beyond the executor’s lifetime.

这个保存shuffle数据的解决办法是用外部的shuffle服务。这个服务需要启动一个长时间运行的进程，其泡在每个集群节点上，独立于spark application和executor之外。如果该服务被允许，executor会从这个服务服务中获取shuffle数据而不是从其他executor中。这意味着，shuffle的状态会被持续保留。

In addition to writing shuffle files, executors also cache data either on disk or in memory. When an executor is removed, however, all cached data will no longer be accessible. There is currently not yet a solution for this in Spark 1.2. In future releases, the cached data may be preserved through an off-heap storage similar in spirit to how shuffle files are preserved through the external shuffle service.

除了写shuffle文件，executor同样会缓存数据在磁盘或者内存中。当executor被移除后，所有cached data将不能被访问。Spark 1.2中并没有解决办法。未来，cached data可能会被保留到off-heep 中，并提供服务。
