---
layout:     post
title:      "Apache Spark Structured Streaming"
subtitle:   "Structured streaming"
date:       2017-12-05
author:     "xp"
header-img: "img/post-bg-universe.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Spark
    - Structed Streaming
---

> From: [Structured streaming](http://www.waitingforcode.com/apache-spark-structured-streaming/structured-streaming/read)

这篇文章主要讨论了由Spark 2.0.0引入的structured streaming概念。第一部分阐述了介绍新的流式处理的原因。第二部分描述了有关新的流式处理的更多细节。最后一部分展示了structured streaming的简单使用。

## Why structured streaming ? 

和许多其他Spark的改进一样（[Spark Project Tungsten][1]也是其中之一），其中之一的**structured streaming**也是来自社区的经验。Spark的贡献者们经常这么做，基于DStreams的streaming有如下问题：

   [1]: http://www.waitingforcode.com/apache-spark-sql/spark-project-tungsten/read

  * incosistent - 通用的批处理（RDD、Dataset）API和流式处理API不一样。当然，在代码层面没有任何问题，但是它应该在处理抽象问题上尽可能简单（尤其在代码维护上）。
  * difficult - 在构建流式管道支持分发策略上比较繁琐：保证精确的一次处理，处理延迟到达的数据或容错。当然，上面提到的都已经实现了，但是使用它们需要程序员做一些额外的工作。

作为上面提出问题的解决方案，structured streaming是一个比较好的解决问题的设计方案。在写作这个文章时（Spark 2.1.0），这个东西仍然是处于实验性阶段。

> 翻译这篇文章时，Spark已发布2.2.0稳定版本，移除实验性标签，即意味着可以用于生产环境了。

## Structured streaming details 

新的流式处理能被理解成一个无界的表，随着新进来的数据而增长，例如，可以认为是构建在[Spark SQL][2]上的流式处理。从而，它能使用Project Tungsten的优化方法和简单的编程模型。感谢在新流式处理在处理方式的改变，structured streaming也减少了实现相关高可用的工作（容错、精确一次处理语义）。

   [2]: http://www.waitingforcode.com/apache-spark-sql

更具体的，structured streaming从Spark借鉴了一些理念 - 至少在项目的某些关键字上是这样：

  * **exactly-once guarantee** (semantic) - structured streaming关注于这个概念上。这就意味着数据仅仅被处理一次并且不会包含重复的输出。
  * **event time** - DStream streaming需要处理的关注的问题之一就是处理的顺序，比如这个例子，数据产生的相对较早而处理的相对较晚。Structured streaming在处理这个问题上使用了event time这个概念，在某些情况下，处理管道允许正确的聚合迟到的数据。
  * **sink** - 编程中的常用的概念，一个sink通常是一个从其他对象或函数中接收事件的类或是函数。Spark使用sinks作为计算结果存储的地方（output sink）。在Spark 2.1中，我们能找到如下的输出sink：file、foreach loop、console或者memory（最后两个sink被标记为调试之用）。
  * **Result Table** - 代表输入数据应用处理逻辑后的结果发送的地方。Result Table的结果直接输出到sink。
  * **output mode** - 在每一个常规的时间间隔后，流处理都会生成一些输出。这些输出在概念上被分组称为输出模式，支持以下输出模式：complete（所有的数据都返回给sink）、append（只有新数据发送给sink）、update（只有更新的数据输出到sink）。
  * **watermark** - 这种方式被用来处理迟到的数据。它跟踪当前的事件时间并且定义一个延迟间隔，这让迟到的事件和已经准时到达的事件能有机会去一起聚合，并且把结果发送到Result Table。可以通过事件窗口来考虑，在超过最大定义时间后将会关闭处理，并且也会关闭对Result Table的返回结果。

为实现上述的改进，structured streaming做了如下的假设：

  * replayable sources - streaming sources应该能在处理失败的情况下重新发送未处理的数据。这意味着他们需要去跟踪数据读取的状态。我们之前提到的Apache Kafka作为可靠的输入源示例，存储了给定分区已经消费的消息偏移量信息。
  * idempotent sinks - output sinks应该能处理重复的写入，例如，不应该增加已经保存的数据。
  * schema - 从streaming是结构化并且是基于DataFrame/Dataset后，很明显，数据必须跟随schema。它简化了很多的处理流程并且保证的类型安全。

## Structured streaming example 

下面的样例代码展示了如何使用structure streaming：
    
    val sparkSession = SparkSession.builder().appName("Structured Streaming test")
      .master("local").getOrCreate()
    
    val csvDirectory = "/tmp/spark-structured-streaming-example"
    
    before {
      val path = Paths.get(csvDirectory)
      val directory = path.toFile
      directory.mkdir
      val files = directory.listFiles
      if (files != null) {
        files.foreach(_.delete)
      }
      addFile(s"${csvDirectory}/file_1.csv", "player_1;3\nplayer_2;1\nplayer_3;9")
      addFile(s"${csvDirectory}/file_2.csv", "player_1;3\nplayer_2;4\nplayer_3;5")
      addFile(s"${csvDirectory}/file_3.csv", "player_1;3\nplayer_2;7\nplayer_3;1")
    }
    
    private def addFile(fileName: String, content: String) = {
      val file = new File(fileName)
      file.createNewFile
      val writer = new BufferedWriter(new FileWriter(file))
      writer.write(content)
      writer.close()
    }
    
    "CSV file" should "be consumed in Structured Spark Streaming" in {
      val csvEntrySchema = StructType(
        Seq(StructField("player", StringType, false),
        StructField("points", IntegerType, false))
      )
    
      // maxFilesPerTrigger = number of files - with that all files will be read at once
      // and the accumulator will store aggregated values for only 3 players
      val entriesDataFrame = sparkSession.readStream.option("sep", ";").option("maxFilesPerTrigger", "3")
        .option("basePath", csvDirectory)
        .schema(csvEntrySchema)
        .csv(csvDirectory)
    
      // Below processing first groups rows by field called 'player'
      // and after sum 'points' property of grouped rows
      val summedPointsByPlayer = entriesDataFrame.groupBy("player")
        .agg(sum("points").alias("collectedPoints"))
    
      val accumulator = sparkSession.sparkContext.collectionAccumulator[(String, Long)]
      val query:StreamingQuery = summedPointsByPlayer.writeStream
        .foreach(new ForeachWriter[Row]() {
          // true means that all partitions will be opened
          override def open(partitionId: Long, version: Long): Boolean = true
    
          override def process(row: Row): Unit = {
            println(s">> Processing ${row}")
            accumulator.add((row.getAs("player").asInstanceOf[String], row.getAs("collectedPoints").asInstanceOf[Long]))
          }
    
          override def close(errorOrNull: Throwable): Unit = {
            // do nothing
          }
        })
        .outputMode("complete")
        .start()
    
      // Shorter time can made that all files won't be processed
      query.awaitTermination(20000)
    
      accumulator.value.size shouldEqual 3
      accumulator.value should contain allOf (("player_1", 9), ("player_2", 12), ("player_3", 15))
    }
    
    case class CsvEntry(player: String, points:Int)
    
    

这篇文章介绍了structured streaming。第一部分阐述了改变的原因。它只是为了标准化抽象数据处理（Dataset/DataFrame代替RDD/DStream）的意愿。第二部分展示了从structured streaming引用而来的一些概念，像exactly-once processing、event time或者output modes。最后一部分呈现了一个由读取CSV文件的structured streaming例子。