---
layout:     post
title:      "Distributed Systems Architecture"
subtitle:   "Spark Memory Management"
date:       2017-07-16
author:     "xp"
header-img: "img/post-bg-digital-native.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Spark
    - Architecture
---

From: <https://0x0fff.com/spark-memory-management/>

从Spark1.6.0版本开始，Spark的内存管理模型已经变了。旧的内存管理模型实现[StaticMemoryManager][1]这个类，现在它被称为“legacy”了。默认“Legacy”模型是被禁用的，这意味着你的代码在Spark 1.5.x和1.6.0上运行将会有不同的行为，一定要小心这个。为了向后兼容，你也可以使用_spark.memory.useLegacyMode_参数开启“legacy”模型，默认它是禁用的

   [1]: https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/memory/StaticMemoryManager.scala

大约在一年之前，我在[article about Spark Architecture][2]这篇文章中阐述过“legacy”的内存管理模型。我也在[Spark Shuffle implementations][3]这篇文章中简短介绍了内存管理模型主题。

   [2]: https://0x0fff.com/spark-architecture/
   [3]: https://0x0fff.com/spark-architecture-shuffle/

这篇文章描述了从Spark 1.6.0版本起的内存管理模型，它实现为[UnifiedMemoryManager][4]。

   [4]: https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/memory/UnifiedMemoryManager.scala

长话短说，新的内存管理模型看起来应该是像这样：

[![Spark Memory Management 1.6.0+][5]][6]

   [5]: https://0x0fff.com/wp-content/uploads/2016/01/Spark-Memory-Management-1.6.0-974x1024.png
   [6]: https://0x0fff.com/wp-content/uploads/2016/01/Spark-Memory-Management-1.6.0.png

Apache Spark统一内存管理器已经在v1.6.0+介绍过了

你可以在图表中看见3个主要的内存区域信息：

  1. **_Reserved Memory_**. 这部分为系统保留内存，并且它的内存大小是固定的。在Spark 1.6.0中，它的值是300MB，这意味着这300MB的内存空间不在Spark内存区域的计算范围之内，它的大小也是没有办法改变的，除非你重新编译Spark源代码或者设置***spark.testing.reservedMemory***这个参数，但我们不建议你这么做，尤其是应用在生产环境中，毕竟它只一个测试参数。注意，这部分的内存只能被称为“reserved”，实现上，它不会以任何方式被Spark使用，但它设置了你使用Spark内存分配的限制。即使你想所所有的Java堆内存都给Spark用来缓存数据，你也不能在这块“reserved”的被称为备用（不是真正的备用，它会存储许多Spark内部的对象）的空间中缓存数据。你所要知道的，如果你不给
Spark executor至少**1.5 * Reserved Memory = 450MB**的堆内存，应用将失败并予以“please use larger heap size”的错误消息提示。
  2. **_User Memory_**. 这部分内存池是在分配***Spark Memory***后剩余的空间，并且这部分内存完全由你喜欢的方式去使用。你可以存储在RDD转换中会用到的所有数据。例如，你可以通过使用mapPartitions转换操作管理hash表重写Spark的aggregation函数，实现数据的聚合操作，这会消耗被称之为_User Memory_的内存。在Spark 1.6.0中，这部分内存池可以用(“_Java Heap_” – “_Reserved Memory_”) * (1.0 – _spark.memory.fraction_)公式来计算，默认它等于 (“_Java Heap_” – 300MB) * 0.25。例如，如果你有4GB的堆内存，那你就有949MB的_User Memory_内存。再次重申一遍，这部分_User Memory_内存存储什么样的内容和怎么使用完全由你而定，Spark完全不会管你在这部分内存中的使用是否超出了内存的界限。不过你自己得小心，以免超过此内存界限而引起OOM错误。
  3. *****Spark Memory*****. 最后，这部分内存池是由Spark自己管理的。它的空间大小可以由(“_Java Heap_” – “_Reserved Memory_”) * _spark.memory.fraction_计算而来，默认在Spark 1.6.0中它的值为(“_Java Heap_” – 300MB) * 0.75。举个例子，若你有4GB的堆内存，那这部分内存的大小就是2847MB。这个池被分成2个区域 - ***Storage Memory***和***Execution Memory***，它们之间的边界由_spark.memory.storageFraction_进行设定，默认值是0.5。新的高级的内存管理模式它们俩的内存边界是不固定的，会因为内存的压力而边界会随之改变。稍后我会讨论这个边界是如何“改变”的，现在让我们聚焦在这部分内存的使用上：

    1. **Storage Memory**. 这个池用来存储Spark的存储数据和序列化后的数据的做“unroll”操作临时展开空间。“广播”变量也会以缓存块的形式存在这个池中。为了防止你的好奇，这里有[unroll][7]的代码。如你所见，它不需要足够的可以内存空间去展开内存块数据，如果期望的持久化方式允许它放在drive中，那么它将未展开的分区直接放在drive中。对于广播变量，它一直是以_MEMORY_AND_DISK_持久化级别保存。
    
    2. **Execution Memory**. 这个池用来存储在执行Spark任务过程中所产生的对象。举个例子，它被用来存储[shuffle intermediate buffer on the Map side][8]，也被用来存储在hash聚合过程中产生的hash表。这个池在没有足够内存可以使用的时候，会将数据溢写到磁盘，但是这里面的块不能被其他的线程（任务线程）强制驱逐出去。

   [7]: https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/storage/MemoryStore.scala#L249
   [8]: https://0x0fff.com/spark-architecture-shuffle/

好，现在让我们聚焦于在***Storage Memory***和***Execution Memory***之间移动的边界。由于***Execution Memory***的特性，你不可以强制从这个池中将内存块数据驱逐出去的，因为这些数据被用来做中间计算的，如果进程需要的这部分内存在它需要的时间找不到，这时候任务就会失败。但这个对***Storage Memory***不是这样子的，它仅仅是在RAM中缓存块数据，并且如果我们从这里面把数据块给驱逐出去，我们只是更新一下存储在RAM中被溢出到磁盘的（或者直接移除）块影响的元数据而已，后续Spark尝试去读取被驱逐时会直接从磁盘读取（在你的持久化策略允许举出到磁盘的话会重新计算所需要的块）。

所以，我们能强制从***Storage Memory***驱逐数据块，但不能在***Execution Memory***中做同样的事。当***Execution Memory***池向***Storage Memory***“借”内存时，它会发生什么？实际情况如下所述：

  * 在***Storage Memory***池中有可用的空间，比如，缓存的数据块没有使用所有可用的内存空间。这时个只是减少***Storage Memory***池的大小，增加***Execution Memory***池的大小。
  * ***Storage Memory***池的大小超过了初始***Storage Memory***区域的大小，并且它已经利用的它所有可以利用的空间，在这种情况下会引发强制驱逐***Storage Memory***池中的内存数据块，直到减小它的大小为它的初始大小。

换句话说，当且仅当在***Execution Memory***池中有多余的可用内存空间时，***Storage Memory***池才能从***Execution Memory***池中“借”内存资源。

你可能还记得***Storage Memory***区域的初始化大小，它的计算方式为“_Spark Memory” * spark.memory.storageFraction =_ (“_Java Heap_” – “_Reserved Memory_”) * _spark.memory.fraction * spark.memory.storageFraction_。它的默认值等于(“_Java Heap_” – 300MB) * 0.75 * 0.5 = (“_Java Heap_” – 300MB) * 0.375。对于4GB的堆内存来说，***Storage Memory***初始区域大小为1423.5MB。

这就意味着如果我们使用Spark缓存数据并且缓存数据的总大小至少和***Storage Memory***区域大小一样的话，我们能保证我们不会驱逐里面的数据使它变得更小。然而，如果你的***Execution Memory***区域在你填充***Storage Memory***之前的增长小于它的初始化值，你就不能将***Execution Memory***中的内存数据块强制驱逐，最终的结果就是你会得到一个比较小的***Storage Memory***区域空间，而其他的空间则被执行过程中的数据块所占据。

我希望这篇文章能帮助你更好的理解Apache Spark内存管理规则，并且通过对内存管理规则的了解来设计你的应用程序。如果你有任务问题，请在评论下方给我留言。