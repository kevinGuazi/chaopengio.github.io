---
layout: post
date: 2016-08-27
title: Build Spark and Test
categories: [spark]
---
### Download spark source code

```bash
git clone git://github.com/apache/spark.git -b branch-1.6
```

### Build spark

- Setting up Maven’s Memory Usage
```bash
export MAVEN_OPTS="-Xmx2g -XX:MaxPermSize=512M -XX:ReservedCodeCacheSize=512m"
```

- Build spark with maven (version >= 3.3.3)
```shell
./make-distribution.sh --tgz -Pyarn -Phadoop-2.6 -Dhadoop.version=2.7.2 -Phive -Phive-thriftserver -Dscala-2.11 -DskipTests
```

```shell
[ERROR] Failed to execute goal net.alchim31.maven:scala-maven-plugin:3.2.2:compile (scala-compile-first) on project spark-sql_2.10: Execution scala-compile-first of goal net.alchim31.maven:scala-maven-plugin:3.2.2:compile failed. CompileFailed -> [Help 1]
[ERROR]
```

Solution:
```shell
./dev/change-scala-version.sh 2.11
```


### Env Dependences

- HADOOP_CONF_DIR
- YARN_CONF_DIR
- SPARK_HOME
- PATH

### Spark access hive error:
```shell
py4j.protocol.Py4JJavaError: An error occurred while calling None.org.apache.spark.sql.hive.HiveContext.
: java.lang.RuntimeException: java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
    at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:522)
    at org.apache.spark.sql.hive.client.ClientWrapper.<init>(ClientWrapper.scala:204)
    at org.apache.spark.sql.hive.client.IsolatedClientLoader.createClient(IsolatedClientLoader.scala:238)
    at org.apache.spark.sql.hive.HiveContext.executionHive$lzycompute(HiveContext.scala:218)
    at org.apache.spark.sql.hive.HiveContext.executionHive(HiveContext.scala:208)
    at org.apache.spark.sql.hive.HiveContext.functionRegistry$lzycompute(HiveContext.scala:462)
    at org.apache.spark.sql.hive.HiveContext.functionRegistry(HiveContext.scala:461)
    at org.apache.spark.sql.UDFRegistration.<init>(UDFRegistration.scala:40)
    at org.apache.spark.sql.SQLContext.<init>(SQLContext.scala:330)
    at org.apache.spark.sql.hive.HiveContext.<init>(HiveContext.scala:90)
    at org.apache.spark.sql.hive.HiveContext.<init>(HiveContext.scala:101)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
    at py4j.reflection.MethodInvoker.invoke(MethodInvoker.java:234)
    at py4j.reflection.ReflectionEngine.invoke(ReflectionEngine.java:381)
    at py4j.Gateway.invoke(Gateway.java:214)
    at py4j.commands.ConstructorCommand.invokeConstructor(ConstructorCommand.java:79)
    at py4j.commands.ConstructorCommand.execute(ConstructorCommand.java:68)
    at py4j.GatewayConnection.run(GatewayConnection.java:209)
    at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
    at org.apache.hadoop.hive.metastore.MetaStoreUtils.newInstance(MetaStoreUtils.java:1523)
    at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.<init>(RetryingMetaStoreClient.java:86)
    at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.getProxy(RetryingMetaStoreClient.java:132)
    at org.apache.hadoop.hive.metastore.RetryingMetaStoreClient.getProxy(RetryingMetaStoreClient.java:104)
    at org.apache.hadoop.hive.ql.metadata.Hive.createMetaStoreClient(Hive.java:3005)
    at org.apache.hadoop.hive.ql.metadata.Hive.getMSC(Hive.java:3024)
    at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:503)
    ... 21 more
Caused by: java.lang.reflect.InvocationTargetException
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
    at org.apache.hadoop.hive.metastore.MetaStoreUtils.newInstance(MetaStoreUtils.java:1521)
    ... 27 more
Caused by: javax.jdo.JDOFatalUserException: Class org.datanucleus.api.jdo.JDOPersistenceManagerFactory was not found.
NestedThrowables:
java.lang.ClassNotFoundException: org.datanucleus.api.jdo.JDOPersistenceManagerFactory
    at javax.jdo.JDOHelper.invokeGetPersistenceManagerFactoryOnImplementation(JDOHelper.java:1175)
    at javax.jdo.JDOHelper.getPersistenceManagerFactory(JDOHelper.java:808)
    at javax.jdo.JDOHelper.getPersistenceManagerFactory(JDOHelper.java:701)
    at org.apache.hadoop.hive.metastore.ObjectStore.getPMF(ObjectStore.java:365)
    at org.apache.hadoop.hive.metastore.ObjectStore.getPersistenceManager(ObjectStore.java:394)
    at org.apache.hadoop.hive.metastore.ObjectStore.initialize(ObjectStore.java:291)
    at org.apache.hadoop.hive.metastore.ObjectStore.setConf(ObjectStore.java:258)
    at org.apache.hadoop.util.ReflectionUtils.setConf(ReflectionUtils.java:76)
    at org.apache.hadoop.util.ReflectionUtils.newInstance(ReflectionUtils.java:136)
    at org.apache.hadoop.hive.metastore.RawStoreProxy.<init>(RawStoreProxy.java:57)
    at org.apache.hadoop.hive.metastore.RawStoreProxy.getProxy(RawStoreProxy.java:66)
    at org.apache.hadoop.hive.metastore.HiveMetaStore$HMSHandler.newRawStore(HiveMetaStore.java:593)
    at org.apache.hadoop.hive.metastore.HiveMetaStore$HMSHandler.getMS(HiveMetaStore.java:571)
    at org.apache.hadoop.hive.metastore.HiveMetaStore$HMSHandler.createDefaultDB(HiveMetaStore.java:624)
    at org.apache.hadoop.hive.metastore.HiveMetaStore$HMSHandler.init(HiveMetaStore.java:461)
    at org.apache.hadoop.hive.metastore.RetryingHMSHandler.<init>(RetryingHMSHandler.java:66)
    at org.apache.hadoop.hive.metastore.RetryingHMSHandler.getProxy(RetryingHMSHandler.java:72)
    at org.apache.hadoop.hive.metastore.HiveMetaStore.newRetryingHMSHandler(HiveMetaStore.java:5762)
    at org.apache.hadoop.hive.metastore.HiveMetaStoreClient.<init>(HiveMetaStoreClient.java:199)
    at org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient.<init>(SessionHiveMetaStoreClient.java:74)
    ... 32 more
Caused by: java.lang.ClassNotFoundException: org.datanucleus.api.jdo.JDOPersistenceManagerFactory
    at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    at java.lang.Class.forName0(Native Method)
    at java.lang.Class.forName(Class.java:348)
    at javax.jdo.JDOHelper$18.run(JDOHelper.java:2018)
    at javax.jdo.JDOHelper$18.run(JDOHelper.java:2016)
    at java.security.AccessController.doPrivileged(Native Method)
    at javax.jdo.JDOHelper.forName(JDOHelper.java:2015)
    at javax.jdo.JDOHelper.invokeGetPersistenceManagerFactoryOnImplementation(JDOHelper.java:1162)
    ... 51 more
```

- reference: [https://community.hortonworks.com/questions/41271/unable-to-run-spark-job-in-clutser-mode.html](https://community.hortonworks.com/questions/41271/unable-to-run-spark-job-in-clutser-mode.html)

1. Check the hive-site.xml contents. Should be like as below for spark.
2. Add hive-site.xml to the driver-classpath so that spark can read hive configuration. Make sure —files must come before you .jar file
3. Add the datanucleus jars using --jars option when you submit
4. Check the contents of hive-site.xml

```xml
 <configuration>
    <property>
      <name>hive.metastore.uris</name>
      <value>thrift://sandbox.hortonworks.com:9083</value>
    </property>
  </configuration>
```

5. Command to run
```shell
spark-submit --master yarn --deploy-mode cluster --py-files spark_config.py \
    --files /usr/local/hive-1.2.1/conf/hive-site.xml \
    --jars $SPARK_HOME/lib/datanucleus-api-jdo-3.2.6.jar,$SPARK_HOME/lib/datanucleus-core-3.2.10.jar,$SPARK_HOME/lib/datanucleus-rdbms-3.2.9.jar \
    pctest.py
```

### Tez jar error
```shell
py4j.protocol.Py4JJavaError: An error occurred while calling None.org.apache.spark.sql.hive.HiveContext.
: java.lang.NoClassDefFoundError: org/apache/tez/dag/api/SessionNotRunning
    at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:529)
    at org.apache.spark.sql.hive.client.ClientWrapper.<init>(ClientWrapper.scala:204)
    at org.apache.spark.sql.hive.client.IsolatedClientLoader.createClient(IsolatedClientLoader.scala:238)
    at org.apache.spark.sql.hive.HiveContext.executionHive$lzycompute(HiveContext.scala:218)
    at org.apache.spark.sql.hive.HiveContext.executionHive(HiveContext.scala:208)
    at org.apache.spark.sql.hive.HiveContext.functionRegistry$lzycompute(HiveContext.scala:462)
    at org.apache.spark.sql.hive.HiveContext.functionRegistry(HiveContext.scala:461)
    at org.apache.spark.sql.UDFRegistration.<init>(UDFRegistration.scala:40)
    at org.apache.spark.sql.SQLContext.<init>(SQLContext.scala:330)
    at org.apache.spark.sql.hive.HiveContext.<init>(HiveContext.scala:90)
    at org.apache.spark.sql.hive.HiveContext.<init>(HiveContext.scala:101)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
    at py4j.reflection.MethodInvoker.invoke(MethodInvoker.java:234)
    at py4j.reflection.ReflectionEngine.invoke(ReflectionEngine.java:381)
    at py4j.Gateway.invoke(Gateway.java:214)
    at py4j.commands.ConstructorCommand.invokeConstructor(ConstructorCommand.java:79)
    at py4j.commands.ConstructorCommand.execute(ConstructorCommand.java:68)
    at py4j.GatewayConnection.run(GatewayConnection.java:209)
    at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.ClassNotFoundException: org.apache.tez.dag.api.SessionNotRunning
    at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    ... 22 more
```

- You need to set the hive engine to mr in _hive-site.xml_
```xml
<property>
  <name>hive.execution.engine</name>
  <value>mr</value>
</property>
```

- New runner.
```shell
spark-submit --master yarn --deploy-mode cluster --py-files spark_config.py \
    --files $SPARK_HOME/conf/hive-site.xml \
    --jars $SPARK_HOME/lib/datanucleus-api-jdo-3.2.6.jar,$SPARK_HOME/lib/datanucleus-core-3.2.10.jar,$SPARK_HOME/lib/datanucleus-rdbms-3.2.9.jar \
    pctest.py
```

```python
from pyspark.sql import SQLContext
sqlContext = SQLContext(sc)
d = [{'Parameters': {'foo': '1', 'bar': '2', 'baz': 'aaa'}}]
df = sqlContext.createDataFrame(d)
```

