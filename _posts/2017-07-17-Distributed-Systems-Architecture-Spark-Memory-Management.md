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

  1. **_Reserved Memory_**. 这部分为系统保留内存，并且它的内存大小是固定的。在Spark 1.6.0中，它的值是300MB，这意味着这300MB的内存空间不在Spark内存区域的计算范围之内，它的大小也是没有办法改变的，除非你重新编译Spark源代码或者设置_spark.testing.reservedMemory_这个参数，但我们不建议你这么做，尤其是应用在生产环境中，毕竟它只一个测试参数。注意，这部分的内存只能被称为“reserved”，实现上，它不会以任何方式被Spark使用，但它设置了你使用Spark内存分配的限制。即使你想所所有的Java堆内存都给Spark用来缓存数据，你也不能在这块“reserved”的被称为备用（不是真正的备用，它会存储许多Spark内部的对象）的空间中缓存数据。你所要知道的，如果你不给
Spark executor至少_1.5 * Reserved Memory = 450MB_的堆内存，应用将失败并予以“please use larger heap size”的错误消息提示。
  2. **_User Memory_**. 这部分内存池是在分配_Spark Memory_后剩余的空间，并且这部分内存完全由你喜欢的方式去使用。你可以存储在RDD转换中会用到的所有数据。例如，你可以通过使用mapPartitions转换操作管理hash表重写Spark的aggregation函数，实现数据的聚合操作，这会消耗被称之为_User Memory_的内存。在Spark 1.6.0中，这部分内存池可以用(“_Java Heap_” – “_Reserved Memory_”) * (1.0 – _spark.memory.fraction_)公式来计算，默认它等于 (“_Java Heap_” – 300MB) * 0.25。例如，如果你有4GB的堆内存，那你就有949MB的_User Memory_内存。再次重申一遍，这部分_User Memory_内存存储什么样的内容和怎么使用完全由你而定，Spark完全不会管你在这部分内存中的使用是否超出了内存的界限。不过你自己得小心，以免超过此内存界限而引起OOM错误。
  3. **_Spark Memory_**. Finally, this is the memory pool managed by Apache Spark. Its size can be calculated as (“_Java Heap_” – “_Reserved Memory_”) * _spark.memory.fraction_, and with Spark 1.6.0 defaults it gives us (“_Java Heap_” – 300MB) * 0.75. For example, with 4GB heap this pool would be 2847MB in size. This whole pool is split into 2 regions – _Storage Memory_ and _Execution Memory_, and the boundary between them is set by _spark.memory.storageFraction_ parameter, which defaults to 0.5. The advantage of this new memory management scheme is that this boundary is not static, and in case of memory pressure the boundary would be moved, i.e. one region would grow by borrowing space from another one. I would discuss the “moving” this boundary a bit later, now let’s focus on how this memory is being used:
    1. **_Storage Memory_**. This pool is used for both storing Apache Spark cached data and for temporary space serialized data “unroll”. Also all the “broadcast” variables are stored there as cached blocks. In case you’re curious, here’s the code of [unroll][7]. As you may see, it does not require that enough memory for unrolled block to be available – in case there is not enough memory to fit the whole unrolled partition it would directly put it to the drive if desired persistence level allows this. As of “broadcast”, all the broadcast variables are stored in cache with _MEMORY_AND_DISK_ persistence level.
    2. **_Execution Memory_**. This pool is used for storing the objects required during the execution of Spark tasks. For example, it is used to store [shuffle intermediate buffer on the Map side][8] in memory, also it is used to store hash table for hash aggregation step. This pool also supports spilling on disk if not enough memory is available, but the blocks from this pool cannot be forcefully evicted by other threads (tasks).

   [7]: https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/storage/MemoryStore.scala#L249
   [8]: https://0x0fff.com/spark-architecture-shuffle/

Ok, so now let’s focus on the moving boundary between _Storage Memory_ and _Execution Memory_. Due to nature of _Execution Memory_, you cannot forcefully evict blocks from this pool, because this is the data used in intermediate computations and the process requiring this memory would simply fail if the block it refers to won’t be found. But it is not so for the _Storage Memory_ – it is just a cache of blocks stored in RAM, and if we evict the block from there we can just update the block metadata reflecting the fact this block was evicted to HDD (or simply removed), and trying to access this block Spark would read it from HDD (or recalculate in case your persistence level does not allow to spill on HDD). 

So, we can forcefully evict the block from _Storage Memory_, but cannot do so from _Execution Memory_. When _Execution Memory_ pool can borrow some space from _Storage Memory_? It happens when either: 

  * There is free space available in _Storage Memory_ pool, i.e. cached blocks don’t use all the memory available there. Then it just reduces the _Storage Memory_ pool size, increasing the _Execution Memory_ pool.
  * _Storage Memory_ pool size exceeds the initial _Storage Memory_ region size and it has all this space utilized. This situation causes forceful eviction of the blocks from _Storage Memory_ pool, unless it reaches its initial size.

In turn, _Storage Memory_ pool can borrow some space from _Execution Memory_ pool only if there is some free space in _Execution Memory_ pool available. 

Initial _Storage Memory_ region size, as you might remember, is calculated as “_Spark Memory” * spark.memory.storageFraction =_ (“_Java Heap_” – “_Reserved Memory_”) * _spark.memory.fraction * spark.memory.storageFraction_. With default values, this is equal to (“_Java Heap_” – 300MB) * 0.75 * 0.5 = (“_Java Heap_” – 300MB) * 0.375. For 4GB heap this would result in 1423.5MB of RAM in initial _Storage Memory_ region. 

This implies that if we use Spark cache and the total amount of data cached on executor is at least the same as initial _Storage Memory_ region size, we are guaranteed that storage region size would be at least as big as its initial size, because we won’t be able to evict the data from it making it smaller. However, if your _Execution Memory_ region has grown beyond its initial size before you filled the _Storage Memory_ region, you won’t be able to forcefully evict entries from _Execution Memory_, so you would end up with smaller _Storage Memory_ region while execution holds its blocks in memory. 

I hope this article helped you better understand Apache Spark memory management principles and design your applications accordingly. If you have any questions, feel free to ask them in comments. 
