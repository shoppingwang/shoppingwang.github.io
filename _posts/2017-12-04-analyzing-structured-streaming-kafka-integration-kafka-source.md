---
layout:     post
title:      "Spark Structured Streaming与Kafka集成分析 - Kafka数据源篇"
subtitle:   "Tuning Spark Applications"
date:       2017-12-04
author:     "xp"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Spark
    - Structed Streaming
    - Kafka
---

> From: [Analyzing Structured Streaming Kafka integration - Kafka source](http://www.waitingforcode.com/apache-spark-structured-streaming/analyzing-structured-streaming-kafka-integration-kafka-source/read)

从这篇文章开始，我们将阐述在Spark Structured Streaming中如何集成Kafka。我们将会回忆在之前讲的source和sink之间的区别，并且会使用代码来展示如何连接至Kafka的broker。在下一个章节中，我们将分析这些代码。第二个章节将会讲解source provider。第三个章节将会专注于offsets的管理。最后一个章节将讲解数据是如何读取的。

## Integrating Kafka with Spark structured streaming 

Structured streaming能集成Kafka作为source和sink，意味着我们能从Kafka里读取数据和写入数据。大体上来说，在Spark structured streaming中集成Kafka的使用主要是配置好参数。这些配置参数定义brokers的地址属性（_bootstrap.servers_），配置数据读取策略参数（参数是startingOffset）和读取的数据源（topic-partition的值对，topics或者topics的正则）。我们也能通过使用重试参数（fetchOffset.numRetries、fetchOffset.retryIntervalMs）很容易的处理失败的情况。

对于写数据来说，还是非常简单的。不过我们也需要去定义bootstrap servers，但这也仅仅是我们必须要定义的唯一一个参数。第二个参数项，topic，它告诉我们数据应该写到哪里去，不过这个参数是可选的。是因为如果我们每条记录都带有topic信息的话，那么我们就不需要去配置全局的topic参数。

下面地使用Kafka structured streaming的简单样例代码：
    
    sparkSession.readStream.format("kafka")
      .options(configuration.getStructuredStreamOptions())
      .load()
    
    

## Kafka source provider 

对于读数据来说，一切始于_org.apache.spark.sql.kafka010.KafkaSourceProvider_这个类。在这个类被调用之前，DataSource类将会使用在DataStreamReader构造过程中调用_format(String)_方法来时指定的信息获取DataSource的provider。这些provider的获取的逻辑是在执行_org.apache.spark.sql.execution.datasources.DataSource#lookupDataSource_方法时发生的，Spark将会使用Java的ServiceLoader类去加载所有实现了_org.apache.spark.sql.sources.DataSourceRegister_接口的类。稍后就会以懒加载的方式迭代处理它们（不会立即处理这些类）。每一个被发现的类都会调用_org.apache.spark.sql.sources.DataSourceRegister#shortName()_方法来获取数据源的短名称，这个值将会与在format(...)方法指定的值进行比较，如果匹配，那么相应的DataSource provider就会被选中。

如果任何已经被加载的类都没有匹配到，Spark会考虑使用format(...)指定的参数值作为类的全路径并尝试直接从classpath中去加载。如果这个类没有被加载到，Spark将会尝试去获取默认的数据源。如你所见，以这种获取方式获取DataSource是非常有用的，你不需要每次在format(...)方法中去定义完整的类名称了。

成功加载source provider类后，将会调用_org.apache.spark.sql.sources.StreamSourceProvider#createSource(org.apache.spark.sql.sources.StreamSourceProvider#createSource)_方法来创建真正的source类对象。

在创建Kafka源过程中，Kafka源的provider定义了如下逻辑：

  * unique group id - 这将允许在单个查询中返回所有的数据。另外，如果第二个查询在和第一个查询一样的topic上执行，两个消费者都会消费部分的数据。这将会创建[消息队列][1]。
  * **driver offset reader** - 很有意思的一个点。Spark将在Driver端会创建一个Kafka的消费者，它只会从Kafka读取数据的偏移量。这个特性在Driver的消费者中不会提交任何的偏移量信息并且它也只会去发现topic的拓扑。整个操作将会返回一个_KafkaOffsetReader_实例。
  * executors configurations - 创建的KafkaOffsetReader和executors的配置后续会传递给KafkaSource。两者的概念和细节会在下一个章节中阐述。

   [1]: http://www.waitingforcode.com/apache-kafka/message-queue-in-apache-kafka/read

## Preparing offsets 

在探索细节实现细节之前，我们先看看下面的这张Spark structured streaming和Kafka模块之间的交互图：

![][2]

   [2]: http://www.waitingforcode.com/public/images/articles/spark_structured_streaming_kafka_integration.png

Spark structured streaming仍然以微批处理的方式在工作。它们是被_org.apache.spark.sql.execution.streaming.StreamExecution_这个类管理的，更具体的说，是由它的实现_org.apache.spark.sql.execution.streaming.StreamExecutionThread#StreamExecutionThread_类来管理的。在内部，它在每指定的时间隔或者只调用一次（OneTimeExecutor）它的私有方法runBatches()来触发数据的获取。在_constructNextBatch()_调用之前，StreamExecutio和**kafka source**会进行一些交互（为第个source获取偏移量信息）。

StreamExecution做的第一件事是和Kafka交互获取偏移量信息，在这个例子中，是返回_(topic partition, offset)_的值对。这些值对是被Kafka source调用_KafkaOffsetReader_中的_fetchLatestOffsets()_方法获取的。这里也就是指定消费偏移量的地方。它将调用KafkaConsumer的一些简单方法（poll、assignment、pause、seekToEnd和position）来获取这些偏移量。

所以返回的构造的分区信息是被StreamExecution的_dataAvailable()_方法用来检测是否有新数据可供读取。是否有新的数据可供读取是由每个分区新的偏移量和当前的偏移量比较得出的。

## Reading data 

如果有新的数据可用，StreamExecution实例会调用_KafkaSource_的_getBatch(start: Option[Offset], end: Offset)_方法来获取两个偏移量之间的数据。之后的第一件事就是通过调用_KafkaSourceOffset_的_getPartitionOffsets(offset: Offset)_方法来转换接收到的偏移量为Map[TopicPartition, Long]实例。 

StreamExecution会跟踪已经消费的偏移量信息。第一个参数（start）在关联在前一个批次中最后的偏移量信息后会发送给getBatch方法。KafkaSource通过使用这个信息来检测新的和被删除的分区信息：
    
    // frompartitionsOffset created from start param
    // untilPartitionsOffset created from end param
    val newPartitions = untilPartitionOffsets.keySet.diff(fromPartitionOffsets.keySet)
    val deletedPartitions = fromPartitionOffsets.keySet.diff(untilPartitionOffsets.keySet)
    
    

当获取需要读取的分区信息后，KafkaSource会创建_KafkaSourceRDDOffsetRange_实例并在稍后会被用作Spark的分区对象的实现转换为Kafka的RDD。值得注意的有趣的一点是，**KafkaSource总是尝试去重复使用已经存在于executors中的Kafka消费者（缓存的）**。由下面的算法实现了上述的功能：
    
    sorted_executors = sort_executors_by_host_and_id(all_executors)
    all_executors = count(sorted_executors)
    for each tp in topic_partitions:
      tp_hash_code = hash_code(tp)
      # below field means the executor's index that should read given
      # topic partition
      prefered_tp_location = tp_hash_code - (tp_hash_code/all_executors) * all_executors
    
    

Spark中做的每一件事最终都会被呈现为一个RDD，KafkaSource会在后面创建一个_KafkaSourceRDD_的实例。它的其他的一些实现和之前计算的过程非常相似：

  * _getPartitions_ - 返回一个由之前创建的KafkaSourceRDDOffsetRange对象到_KafkaSourceRDDPartition_分区信息的映射列表。
  * _getPreferredLocations(split: Partition)_ - KafkaSourceRDDPartition是一个case class，它包含两个字段：index: Int, offsetRange: KafkaSourceRDDOffsetRange。这个方法直接使用offsetRange的值来返回Kafka指定topic分区中已经定义的executors信息。
  * _compute(thePart: Partition, context: TaskContext)_ - 在RDD被创建的同时_reuseKafkaConsumer_标记位被设置为true。这也就是为什么它将尝试去获取已经存在的Kafka消费者来从特定的分区中读取数据的原因。

通过调用_CachedKafkaConsumer_的方法next()在NextIterator[ConsumerRecord[Array[Byte]中来返回分区的记录信息，内部实现上，是使用KafaConsumer通过传递偏移量信息来读取数据。读取数据由下面两步组成：

  1. 为给定的分区搜索指定的偏移量信息（_org.apache.kafka.clients.consumer.KafkaConsumer#seek(TopicPartition partition, long offset)_）
  2. 通过CachedKafkaConsumer来拉取分区数据（_org.apache.kafka.clients.consumer.KafkaConsumer#poll(long timeout)_）

数据获取的一致性是被严格控制的。每次当检测到偏移量信息不一致时，例如，读取的偏移量信息大于最大的可用偏移量信息，CachedKafkaConsumer要么会抛出一个IllegalStateException异常，或者记录一个类似于“Skip missing records in ....”的警告日志。

这篇文章描述了Kafka和Spark structured streaming之间的集成信息。第一部分回顾了一点sources和sinks的概念。第二部分展示了通过format(...)方法接收类全名称或它的别名定义streaming source的两种方式。接下来的部分展示了Spark是如何使用在driver中创建的偏移量相关的KafkaConsumer实例来处理Kafka的偏移量信息。最后一部分展示了数据是如何读取的。我们可能已经看到Spark尝试通过应用指定的算法来重复使用已经创建的consumer。我们也能观测到每次控制的消费偏移量信息不一致被检测到时，会抛出异常或记录一个警告信息。