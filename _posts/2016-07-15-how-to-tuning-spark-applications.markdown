---
layout:     post
title:      "Spark应用程序调优"
subtitle:   "Tuning Spark Applications"
date:       2016-07-15
author:     "xp"
header-img: "img/post-bg-unix-linux.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Spark
---

> 原文连接[Tuning Spark Applications](http://www.cloudera.com/documentation/enterprise/latest/topics/admin_spark_tuning1.html)。
>
> 文中措词未加详细斟酌，望见谅。


这个主题描述了各方面Spark应用程序调优的方法。在调试的过程中，你应该同时监控你的应用程序的行为以便知晓调优操作的结果是否有效。

更多的关于Spark应用程序监控的方法，详情[Monitoring Spark Applications](http://www.cloudera.com/documentation/enterprise/latest/topics/operation_spark_applications.html#concept_qb3_pfd_3s)。

继续阅读：

* This will become a table of contents (this text will be scraped).
{:toc}

### Shuffle Overview
单个的Spark数据集由固定数量的分区组成，每一个分区由一些记录组成。对于返回**窄依赖**的数据集转换操作，比如`map`、`filter`，在一个单一的分区中这些记录的计算依赖单一的父分区数据集。每一个对象都只依赖于单独的父对象。某些操作例如`coalesce`将会导致在同一个任务处理过程中处理多个输入分区，但是这样的转换依赖是窄依赖，因为用来计算单一任务的输出的输入的记录记录仍然只依赖于有限的分区子集。

Spark也支持**宽依赖**的转换操作，比如`groupByKey`、`reduceByKey`。在这些依赖中，用来在单个分区里计算结果集的数据会依赖于*许多父分区*数据集。为了执行这些转换操作，所有的拥有相同键值的元组最终会被送入同一个分区当中，并且被同一个任务处理。为了满足这样的需求，Spark执行*shuffle*操作，会在集群里传输数据并且把新的[stage](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_spark_apps.html#concept_ar2_1rc_w5)生成的结果以分区集合的方式呈现。

例如，考虑如下的代码：

```scala
sc.textFile("someFile.txt").map(mapFunc).flatMap(flatMapFunc).filter(filterFunc).count()
```
它在一个被分割了的文本文件上执行一系列的转换操作后执行了单一的`count`操作，这段代码会在一个单一的stage中运行，因为这些转换操作的输出不依赖于其他分区作为自己的输入。

相对的，下面的Scala代码将会计算所有单词的哪些字母在文本文件中出现的次数超过1000次：

```scala
val tokenized = sc.textFile(args(0)).flatMap(_.split(' '))
val wordCounts = tokenized.map((_, 1)).reduceByKey(_ + _)
val filtered = wordCounts.filter(_._2 >= 1000)
val charCounts = filtered.flatMap(_._1.toCharArray).map((_, 1)).reduceByKey(_ + _)
charCounts.collect()
```
这个例子有三个stage。两个`reduceByKey`操作触发了stage的边界，因为计算他们的输出需要重新对数据按照键值进行分区。

最后一个例子是一个相对复杂的转换操作图谱，包括了多依赖的`join`操作：
![](http://www.cloudera.com/documentation/enterprise/latest/images/xspark-tuning-f2.png.pagespeed.ic.PBJeNuYS65.png)

下面粉红色的方框代表的运行的stage图：
![](http://www.cloudera.com/documentation/enterprise/latest/images/xspark-tuning-f3.png.pagespeed.ic.Sdhu8hhAlr.png)

在每一个stage的边界，数据在父stage中被任务写入到磁盘，然后被子stage通过网络把数据取走。因为这样会导致磁盘频繁高度和频繁的网络I/O，因为stage的边界操作代价比较大，所以在操作的时候尽可能避免使用。在父stage里的分区的数量和子stage里分区的数量可能不同。转换操作能触发stage边界，典型的是转换接受一个`numPartitions`参数，这个参数指定了将会有多少分区将会被划分至子stage。和MapReduce 作业一样重要的reducers数量参数，stage边界分区的数量决定了应用程序的性能。[调整分区的数量](http://www.cloudera.com/documentation/enterprise/latest/topics/admin_spark_tuning1.html#concept_pqk_tfs_gs__section_qsh_g2t_b5)也就是如何去调整这个参数。

### Choosing Transformations to Minimize Shuffles
你通常可以从一系列的转换和动作中选择你想要的从而产生相同的结果。然而，不是所有的得到这些结果有着相同的性能。避免觉的陷阱并且选择合适的操作能明显的提高应用程序的性能。

当你 选择一系列的操作时，尽量减少shuffle操作次数和shuffle操作的数据量。Shuffle操作代价是昂贵的；所有的shuffle数据必须写入磁盘然后通过网络进行传输。`repartition`、`join`、`cogroup`和任何的以`*By`或者`*ByKey`结尾的操作都会引发shuffle操作。不是所有的这些操作都是等同的，然而，你需要避免如下的使用方式：

* `groupByKey` 执行一个联合的减少操作。例如，rdd.groupByKey().mapValues(_.sum)会产生和rdd.reduceByKey(_ + _)一样的结果。然而，当为每一个分区的每一个键本地计算sum前，前一个转换操作会将整个数据集通过网络传输，进行shuffle操作后组合这些本地的sum成一个更大的sum。
* `reduceByKey` 输入和输出的结果类型不一样。例如，考虑写一个转换操作，这个操作将找出所有的键关联的唯一字符串。你应该使用`map`去转换每一个元素放到一个`Set`中然后用`reduceByKey`结合`Set`的结果：
```scala
rdd.map(kv => (kv._1, new Set[String]() + kv._2)).reduceByKey(_ ++ _)
```
这些结果创建了不必要的一些对象，因为每一个记录分配了一个新的set。

相反，使用`aggregateByKey`操作，将会在map端进行聚合会更有效率：

```scala
val zero = new collection.mutable.Set[String]()
rdd.aggregateByKey(zero)((set, v) => set += v,(set1, set2) => set1 ++= set2)
```
* `flatMap-join-groupBy` 当两个数据集已经按键进行分组，你想联合他们并且保持他们在同一个组中，使用`cogroup`操作。这样避免了额外的解包和打包分组操作。

### When Shuffles Do Not Occur
在某些情况下，上面描述的转换*不会*引发shuffle操作。Spark不会触发shuffle操作当前一个转换操作已经把数据分在*同一个分区*里。考虑以下的流程：

```scala
rdd1 = someRdd.reduceByKey(...)
rdd2 = someOtherRdd.reduceByKey(...)
rdd3 = rdd1.join(rdd2)
```
因为没有partitioner传递给`reduceByKey`，所以将会使用默认的partitioner，结果是在`rdd1`和`rdd2`都进行了哈希分区。这两个`reduceByKey`转换结果引发了两个shuffle操作。如果数据集有相同的分区数量，那么join将不需要额外的shuffle操作。因为数据集被一致的进行分区，任何在`rdd1`中单独的分区所拥有的键将只会出现在同一个`rdd2`有分区中。因此，`rdd3`的单个分区输出仅仅只依赖于单个`rdd1`和单个`rdd2`的分区，并且第三个shuffle操作是不需要的。

例如，如果某个rdd有四个分区，某个其他rdd有两个分区，并且两个reduceByKeys操作使用三个分区，那么运行的任务数据看起来会像下面这样：
![](http://www.cloudera.com/documentation/enterprise/latest/images/spark-tuning-f4.png)

如果`rdd1`和`rdd2`使用不同的分区策略或者使用默认的分区策略但使用不同的分区数，当做join操作时只有一个数据集（较少分区的那个）要被重新做shuffle操作：
![](http://www.cloudera.com/documentation/enterprise/latest/images/spark-tuning-f5.png)

为了避免在连接两个数据集进行shuffle操作，你可以使用[广播变量](https://spark.apache.org/docs/1.6.0/programming-guide.html#broadcast-variables)。当其中一个数据足够小到能放到单个executor内存中，它能在driver端被加载进一个hash表并且被广播到每一个executor。一个map转换然后就能在查找时引用到这个hash表。

### When to Add a Shuffle Transformation
上面最小化shuffle的数量规则有一些例外的地方。

一个额外的shuffle操作能增加并行度。例如，如果你的数据来自一些大的未分割的文件，由`InputFormat`产生的每一个分区可能包含许多记录，这样就没有足够多的分区去使用所有可用的核心。在这个例子中，当加载数据时使用一个较高数量的分区数（触发shuffle）调用repartition操作，将允许接下来的转换操作使用更多的集群CPU。

另一个例子出现在当使用`reduce`和`aggregate`动作在driver中聚合数据。当使用一个较高分区数进行聚合时，在driver端一个单独的线程中把所有的结果合并在一起计算很快会成为瓶颈。为了减轻driver端的负载，首先使用`reduceByKey`或者`aggregateByKey`执行一系列的分布的聚合以便将数据集分割成一些小的分区。在将这些数据送入driver做最后一轮的合并之前，这些值在每一个分区里进行并行的单独的合并。查看[treeReduce](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/rdd/RDD.scala#L1030)和[treeAggregate](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/rdd/RDD.scala#L1104)的例子是如何操作的。

这个方法非常有用当聚合操作已经是按键进行分组了的时候。例如，考虑一个应用计算每一个单词在文集中出现的数量，并把计算的结果作为一个map放入driver中。一种方法，使用`aggregate`操作能够完成操作，每一个分区是一个本地map计算，然后在driver端合并map结果。另一种方法，使用`aggregateByKey`方法能完全分布式的执行完成任务，然后在driver端简单的使用`collectAsMap`获取结果。

### Secondary Sort
[repartitionAndSortWithinPartitions](https://spark.apache.org/docs/1.6.0/api/scala/#org.apache.spark.rdd.OrderedRDDFunctions)转换操作会通过一个分区器重新对数据集进行分区，并且针对每一个分区结果，将会按它们的键顺序进行排序。这个转换将排序操作下放到排序机器上，所以大的数据能够被有效的分发并且排序能够和其他的操作一起执行。

例如，Apache Hive on Spark在它的`join`实现中使用这种转换。在[第二种排序](http://www.quora.com/What-is-secondary-sort-in-Hadoop-and-how-does-it-work)模式中构建block它也扮演着至关重要的角色，你会将你的数据使用键进行分组，并且使用键对值进行迭代操作，使它们表现出局部有序。这种场景出现在需要对基于时间发生的事件按用户进行分组并且对每个用户分析这些事件的算法中。

### Tuning Resource Allocation
如何使Spark应用程序使用YARN集群管理器，更多的信息请参考[Running Spark Applications on YARN](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_running_spark_on_yarn.html#concept_b3g_nxy_pr)。

最主要的两个被Spark和YARN管理的资源是CPU和内存。磁盘和网络I/O对Spark的性能也有影响，但是既不是Spark也不是YARN管理它们。

每一个Spark应用中的执行器都有相同固定数量的核数和固定的堆内存。使用`--executor-cores`命令标记指定核的数量，或者设置`spark.executor.cores`属性。相似的，使用`--executor-memory`命令标记控制堆内存的大小或者设置`spark.executor.memory`属性。指定`核`的属性控制了当前任务在executor上的并发数量。例如，为每一个executor设置`--executor-cores 5`时最大能有5个任务同时在运行。内存的属性控制着Spark能缓存的数据总量，除开最大的用来分组、聚合、联接的shuffle数据结构。

从CDH 5.5开始，[动态分配](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_running_spark_on_yarn.html#concept_zdf_rbw_ft)就是被开启的，它将会动态的增加或者移除executor。为了显式的控制executor的数量，你可以使用`--num-executors`命令行标记或者使用`spark.executor.instances`配置参数进行参数值覆写。

考虑Spark在YARN中申请多少资源是已经可用的。相关的YARN属性如下：

* `yarn.hostmanager.resource.memory-mb` 控制每个主机中container能使用的最大内存总和。
* `yarn.hostmanager.resource.cpu-vcores` 控制每个主机中container能使用的最大CPU核心数。

请求五个执行核心在YARN中就是请求五个核。内存的申请在YARN中更复杂，基于以下两个原因：

* `--executor-memory`/`spark.executor.memory`属性控制着堆内存的大小，但是JVM能够使用一些堆外内存，比如拘留字符串和直接分配字节缓冲区。`spark.yarn.executor.memoryOverhead`属性的值被增加到executor的内存中来决定在YARN中需要申请的所有内存。默认值是`max`(384, .07 * `spark.executor.memory`)。
* YARN可能周期的慢慢申请内存。`yarn.scheduler.minimum-allocation-mb`和`yarn.scheduler.increment-allocation-mb`属性各自控制最小的和增加的申请值。

下面的图表（未考虑衡量默认值）展示了在Spark和YARN中内存的继承信息：
![](http://www.cloudera.com/documentation/enterprise/latest/images/spark-tuning2-f1.png)

在估计Spark executor大小时记住如下原则：

* ApplicationMaster不是一个能从YARN中申请container的executor容器，需要的内存和CPU也必须计算在内。在**客户端**部署模式中，它们的默认值是1024 MB和一个核心。在**集群**部署模式中，ApplicationMaster运行driver，所以考虑使用`--driver-memory`和`--driver-cores`来获取它需要的资源。
* 使用过大的内存运行executor会导致过长的垃圾收集延迟。单个executor最大的内存上限为64GB。
* HDFS客户端很难去处理很多的并发线程。最多，每个executor五个任务就能达到满写负载，所以保持每个executor的核心数小于那个值。
* 运行微小的executor（举例说只有一个核并且只有足够的内存跑一个任务）抵消了在单个JVM里运行多个任务的好处。例如广播变量必须复制到每一个executor，如此多的executor将会导致更多的数据拷贝。

### Resource Tuning Example
考虑有一个集群运行着6个NodeManager，每一个都是拥有16核和64GB内存。

NodeManager的容量，`yarn.nodemanager.resource.memory-mb`和`yarn.nodemanager.resource.cpu-vcores`应该各自设置为63 * 1024 = 64512 (megabytes)和15。避免分配100%的资源给YARN容器，因为主机需要一些资源去跑OS和Hadoop的守护进程。在这种情况下，保留1GB和1个核心去处理系统进程。Cloudera Manager自动的计算这些并且配置这些YARN属性。

你可能考虑使用`--num-executors 6 --executor-cores 15 --executor-memory 63G`。然而，这种方式行不通：

* 63 GB加上executor额外的内存已经超出了NodeManager 63 GB 的内存容量。
* ApplicationMaster使用主机上的一个核，所以主机这里就没有15个核心executor空间。
* 每个executor15个核心将会导致不好的HDFS I/O吞吐量。

相反，使用`--num-executors 17 --executor-cores 5 --executor-memory 19G`：

* 这样做的结果是在所有的主机上有三个executor，除了其中一个有ApplicationMaster和两个executors。
* `--executor-memory` 计算结果为(63/3 executors per host) = 21. 21 * 0.07 = 1.47. 21 - 1.47 ~ 19。

### Tuning the Number of Partitions
Spark已经限制了达到最优的并行容量。每一个Spark stage有一定数量的任务，每一个进程顺序的处理数据。每个stage的任务的数量是决定性能的最重要的参数。

和[Spark Execution Model](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_spark_apps.html#concept_ar2_1rc_w5)描述的一样，Spark在stage中分组数据集。在stage中任务的数量和最后一个stage中分区的数量是一样的。数据集的分区数量和它依赖的数据集分区数量是一样的，除了以下情况以外：
* `coalesce`操作创建比它的父数据集更少的数据集分区。
* `union`操作创建和它父数据集分区数量和一样多的数据集分区。
* `cartesian`操作创建和它父数据集输出分区数量的分区。

像由`文本文件`或者`Hadoop文件`产生的数据集没有父亲，它们的分区数据由底层的MapReduce `InputFormat`决定。典型的，读取每一个HDFS块将会产生一个分区。数据集的分区数可以由`parallelize`方法指定，若方法未指定则由`spark.default.parallelism`控制。为了决定一个数据集的分区数量，调用`rdd.partitions().size()`。

如果需要运行任务的数量小于小于可用的槽数，CPU的使用是不理想的。除此之外，聚合操作使用更多的内存在每个任务中也会发生。在`join`、`cogroup`、或者`*ByKey`操作中，对象被hashmap或者在内存缓冲中持有并分组或者排序。在任务的stage中`join`、`cogroup`和`groupByKey`使用这些数据将会触发获取数据时的shuffle操作。在任务的stage中`reduceByKey`和`aggregateByKey`使用这些数据将会触发两端的shuffle操作。如果这些记录在聚合的操作中需要额外的内存，可能会发生下面的问题：
* 在这些数据结构中持有大量的记录会增加垃圾回收压力，也将会导致计算的暂停。
*  Spark写出这些数据到磁盘，导致磁盘I/O和排序引发任务暂停。

为了增加stage从Hadoop读取数据的分区数，你可以按如下方式去做：

* 使用`repartition`转换操作将会触发shuffle。
* 配置你的`InputFormat`来创建更多的分块。
* 写入HDFS的数据使用更小的数据块大小。

如果当前stage正在接收其他stage的数据，转换操作接收一个numPartitions参数将会触发stage边界：
```scala
val rdd2 = rdd1.reduceByKey(_ + _, numPartitions = X)
```
决定最优的X值需要进行试验。找出父数据集的分区数量，并且乘以1.5倍直到性能停止提升。

你也可以更公式化的做法去计算X的值，但是某些变量用公式是 很难去求的。我们的主要目的是运行足够多的任务以便数据去往每个任务的数据适合它的可用内存。每个任务可用内存如下：
```scala
(spark.executor.memory * spark.shuffle.memoryFraction * spark.shuffle.safetyFraction)/spark.executor.cores
```
`memoryFraction`和`safetyFraction`各自默认值是0.2和0.8。
数据shuffle需要的内存量更难去衡量。最接近探索的方式是找到一个运行过的stage中shuffle操作写到内存和写到磁盘比率。然后，用这个数乘以所以的shuffle写入的总数。然而，如果stage正在执行一个裁减操作，这将会是更为复杂的：
```scala
(observed shuffle write) * (observed shuffle spill memory) * (spark.executor.cores)/(observed shuffle spill disk) * (spark.executor.memory) * (spark.shuffle.memoryFraction) * (spark.shuffle.safetyFraction)
```
然后，慢慢的往上调，因为更多的分区往往比更少的分区有更好的性能。
当有问题时，许多的任务执行时将出现错误（因此需要分区）。推荐和MapReduce进行对比，它不像Spark，任务有着高启动开销。

### Reducing the Size of Data Structures
在Spark中，数据流是以记录的方式进行处理。一条记录有两种表现形式：一种是反序列化的Java对象，一种是序列化的二进制。通常来说，Spark使用反序列化在内存中代表记录，而当存储至磁盘或者通过网络传输时则进行序列化。对于基于排序的shuffle，内存数据排序被存储为序列化形式。

`spark.serializer`属性控制这两种表现形式之间的序列化。Cloudera建议使用Kryo序列化`org.apache.spark.serializer.KryoSerializer`。

在你的这两种形式数据记录方式对Spark的性能有很大的影响。回顾已有的数据类型并且找出一种方式来减小它们的大小。大的反序列化对象会引发Spark更频繁的将数据溢出到磁盘中，并且使能够缓存（例如`MEMORY`存储策略）在Spark中的反序列化记录数更少。Apache Spark调优指引描述了怎么去[减小这些对象的大小](https://spark.apache.org/docs/1.6.0/tuning.html#memory-tuning)。大的序列化的对象将导致需要更多的磁盘和网络I/O，并且也使能够缓存（例如`MEMORY_SER`存储策略）在Spark中的反序列化记录数更少。确认[SparkConf#registerKryoClasses](https://spark.apache.org/docs/1.6.0/api/scala/index.html#org.apache.spark.SparkConf) API使用注册任意的你使用的自定义类。

### Choosing Data Formats
当你保存数据至硬盘时，使用可扩展的二进制格式，例如Avro、Parquet、Thrift或者Protobuf并且存储进[sequence文件](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/SequenceFile.html)中。