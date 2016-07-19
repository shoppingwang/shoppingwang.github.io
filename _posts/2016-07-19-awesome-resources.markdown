---
layout:     post
title:      "Awesome Resources"
subtitle:   "A list of awesome frameworks, libraries and software."
date:       2016-07-19
author:     "xp"
header-img: "img/post-bg-miui6.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Resource
---
* This will become a table of contents (this text will be scraped).
{:toc}


## Elasticsearch

### [Elasticsearch权威指南（中文版）](https://www.gitbook.com/book/looly/elasticsearch-the-definitive-guide-cn/details)
----
Elasticsearch权威指南（中文版） — Golden Looly


## Git

### [闯过这 54 关，点亮你的 Git 技能树](https://www.v2ex.com/t/247270)
--
如今， Git 大行其道，颇有一统天下之势。
如果你的技能树上 Git 和 Github 的图标还没有点亮的话，你都不好意思说你是程序员。
别说互联网企业，我接触到的许多传统企业都在从 SVN ， Clear Case 等迁移到 Git 上，甚至大厂还会有一个团队去定制适合自己企业的 Git 服务器。

很多人简历上写的「精通 Git 与 Github 」，但如果你问他熟悉到什么程度的话，回答通常是「就是会用常用的 add，commit，push 操作」。


### [GitHub Pages 指南](http://wiki.jikexueyuan.com/project/github-pages-basics/)
--
GitHub Pages 可以为你或者你的项目提供介绍网页，它是由 GitHub 官方托管和发布的。你可以使用 GitHub 提供的页面自动生成器，GitHub App 或者命令行来创建 GitHub Pages。

本指南是 GitHub Pages 官网 [GitHub Pages Basics](https://help.github.com/categories/github-pages-basics/) 的中文翻译版本。


### [Git下载单个文件夹](http://kinolien.github.io/gitzip/)
--
It can make sub-folder/sub-directory of github repository as zip and download it.


## Java

### [Awesome Java](https://github.com/akullpp/awesome-java)
--
A curated list of awesome Java frameworks, libraries and software.


### [Swiss Java Knife](https://github.com/aragozin/jvm-tools)
--
SJK is a command line tool for JVM diagnostic, troubleshooting and profiling.

SJK exploits standard diagnostic interfaces of JVM (such as JMX, JVM attach and perf counters) and add some more logic on top to be useful for common troubleshooting case.


### [Automon](https://github.com/stevensouza/automon)
--
Automon combines the power of AOP (AspectJ) with monitoring tools (JAMon, JavaSimon, Yammer Metrics, StatsD, ...) or logging tools (perf4j, log4j, sl4j, ...) that you already use to declaratively monitor the following:

- Your Java code,
- The JDK,
- Any jars used by your application

Automon is typically used to track method invocation time, and exception counts. It is very easy to set-up and you should be able to start monitoring your code within minutes. The data will be stored and displayed using the monitoring tool of your choice. The following image shows the type of data Automon collects (The example below displays the data in JAMon, however the data can be displayed in whatever monitoring tool/api you choose. For example here is same data displayed in [grahphite/StatsD](https://github.com/stevensouza/automon/blob/master/docs/automon_statsd.png)).


### [Ninety-Nine Problems in Java 8, Scala, and Haskell](https://github.com/shekhargulati/99-problems)
--
This is an adaptation of the [Ninety-Nine Prolog Problems](https://sites.google.com/site/prologsite/prolog-problems) written by Werner Hett at the Berne University of Applied Sciences in Berne, Switzerland.

From the original source:

>The purpose of this problem collection is to give you the opportunity to practice your skills in logic programming. Your goal should be to find the most elegant solution of the given problems. Efficiency is important, but logical clarity is even more crucial. Some of the (easy) problems can be trivially solved using built-in predicates. However, in these cases, you learn more if you try to find your own solution.

>The problems have different levels of difficulty. Those marked with a single asterisk (\*) are easy. If you have successfully solved the preceding problems you should be able to solve them within a few (say 15) minutes. Problems marked with two asterisks (\*\*) are of intermediate difficulty. If you are a skilled programmer it shouldn't take you more than 30-90 minutes to solve them. Problems marked with three asterisks (***) are more difficult. You may need more time (i.e. a few hours or more) to find a good solution.


### [ND4J: N-Dimensional Arrays for Java](http://nd4j.org)
--
ND4J and ND4S are scientific computing libraries for the JVM. They are meant to be used in production environments, which means routines are designed to run fast with minimum RAM requirements.

**Main features**

- Versatile n-dimensional array object
- Multiplatform functionality including GPUs
- Linear algebra and signal processing functions

A usability gap has separated Java, Scala and Clojure programmers from the most powerful tools in data analysis, like NumPy or Matlab. Libraries like Breeze don’t support n-dimensional arrays, or tensors, which are necessary for deep learning and other tasks. ND4J and ND4S are used by national laboratories for tasks such as climatic modeling, which require computationally intensive simulations.

ND4J brings the intuitive scientific computing tools of the Python community to the JVM in an open source, distributed and GPU-enabled library. In structure, it is similar to SLF4J. ND4J gives engineers in production environments an easy way to port their algorithms and interface with other libraries in the Java and Scala ecosystems.


### [Chronicle Map](https://github.com/OpenHFT/Chronicle-Map)
--
Chronicle Map is an in-memory key-value store designed for low-latency and/or multi-process applications. Notably trading, financial market applications.

**Features**

- **Ultra low latency**: Chronicle Map targets median latency of both read and write queries of less than 1 microsecond in certain tests.
- **High concurrency**: write queries scale well up to the number of hardware execution threads in the server. Read queries never block each other.
- (Optional) **persistence to disk**
- **Multi-key queries**
(Optional, closed-source) eventually-consistent, fully-redundant, asynchronous replication across servers, "last write wins" strategy by default, allows to implement custom state-based CRDT strategy.

**Unique features**

- Multiple processes could access a Chronicle Map concurrently. At the same time, the data store is in-process for each of the accessing processes. (Out-of-process approach to IPC is simply incompatible with Chronicle Map's median latency target of < 1 μs.)

- Replication without logs, with constant footprint cost, guarantees progress even if the network doesn't sustain write rates.


### [LeakCanary](https://github.com/square/leakcanary)
--
A memory leak detection library for Android and Java.


### [Gumshoe Load Investigator](https://github.com/dcm-oss/gumshoe)
--
Monitor application performance statistics associated with individual calling stacks, interactively filter and view as a flame graph or root graph.

Gumshoe was first created initially for internal use in the Dell Cloud Manager application but source code has since been released for public use under [these terms](https://github.com/dcm-oss/gumshoe/blob/master/COPYRIGHT).


### [Bootique](https://github.com/nhl/bootique)
--
Bootique is a [minimally opinionated](https://medium.com/@andrus_a/bootique-a-minimally-opinionated-platform-for-modern-java-apps-644194c23872#.odwmsbnbh) technology for building container-less runnable Java applications. With Bootique you can create REST services, webapps, jobs, DB migration tasks, etc. and run them as if they were simple commands. No JavaEE container required! Among other things Bootique is an ideal platform for Java microservices, as it allows you to create a fully functional app with minimal setup.


### [Dex](https://github.com/PatMartin/Dex)
--
Dex : The data explorer is a data visualization tool written in Java/JavaFX capable of powerful ETL and data visualization.


### [strman-java](https://github.com/shekhargulati/strman-java)
--
A Java 8 library for working with String. It is inspired by [dleitee/strman](https://github.com/dleitee/strman).


### [ReactiveX文档中文翻译](https://www.gitbook.com/book/mcxiaoke/rxdocs/details)
--
ReactiveX和RxJava文档中文翻译 — mcxiaoke


### [Awesome-RxJava](https://github.com/lzyzsd/Awesome-RxJava)
RxJava resources.



## Oozie

### [Apache Oozie Offical Document Study（中文）](http://oozie-study.readthedocs.io/en/latest/)
--
This is my study paper of Oozie offical document, you can refer http://oozie.apache.org for more information.

Oozie是用来管理 Apache Hadoop 作业的工作流调度系统.

Oozie Workflow 作业是一个 DAG (Directed Acyclical Graph) 活动.

Oozie Coordinator作业 由 时间(周期)或 有效数据 触发执行.

Oozie 支持多种Hadoop作业类型(例如Java map-reduce, Streaming map-reduce, Pig, Hive, Sqoop and Distcp)以及系统特定作业(例如Java 程序和shell脚本) 的集成.

Oozie是一个大规模,高可靠,可扩展工作流调度系统.



## Scala

### [Effective Scala中文版](http://twitter.github.io/effectivescala/index-cn.html)
--
这是一篇“活的”文档，我们会更新它,以反映我们当前的最佳实践，但核心的思想不太可能会变： 永远重视可读性；写泛化的代码但不要牺牲清晰度； 利用简单的语言特性的威力，但避免晦涩难懂（尤其是类型系统）。最重要的，总要意识到你所做的取舍。一门成熟的(sophisticated)语言需要复杂的实现，复杂性又产生了复杂性：之于推理，之于语义，之于特性之间的交互，以及与你合作者之间的理解。因此复杂性是为成熟所交的税——你必须确保效用超过它的成本。

玩的愉快。


### [Scala 课堂!](http://twitter.github.io/scala_school/zh_cn/index.html)
--
Scala课堂是Twitter启动的一系列讲座，用来帮助有经验的工程师成为高效的Scala 程序员。Scala是一种相对较新的语言，但借鉴了许多熟悉的概念。因此，课程中的讲座假设听众知道这些概念，并展示了如何在Scala中使用它们。我们发现这是一个让新工程师能够快速上手的有效方法。网站里的是伴随这些讲座的书面材料，这些文字材料本身也是很有用的。


### [Scala文档大集合](http://code.csdn.net/scala/)
--
Scala是一种多范式的编程语言，其设计的初衷是要集成面向对象编程和函数式编程的各种特性。Scala运行于Java平台（Java虚拟机），并兼容现有的Java程序。它也能运行于CLDC配置的Java ME中。目前还有另一.NET平台的实现，不过该版本更新有些滞后。Scala的编译模型（独立编译，动态类加载）与Java和C#一样，所以Scala代码可以调用Java类库（对于.NET实现则可调用.NET类库）。Scala包括编译器和类库，以BSD许可证发布。2009年4月，Twitter宣布他们已经把大部分后端程序从Ruby迁移到Scala，其余部分也打算要迁移。此外，Wattzon已经公开宣称，其整个平台都已经是基于Scala基础设施编写的。瑞银集团也把Scala用于一般产品中。


### [Scala概览](http://docs.scala-lang.org/zh-cn/overviews/)
--
Scala主要特性介绍。


### [Scala Tour](http://docs.scala-lang.org/tutorials/tour/tour-of-scala.html)
--
Scala is a modern multi-paradigm programming language designed to express common programming patterns in a concise, elegant, and type-safe way. It smoothly integrates features of object-oriented and functional languages.


## Spark
### [Kafka-Spark-Consumer](https://github.com/dibbhatt/kafka-spark-consumer)
--
High Performance Kafka Consumer for Spark Streaming.


### [Spark机器学习算法研究和源码分析](https://github.com/endymecy/spark-ml-source-analysis)
--
本项目对spark ml包中各种算法的原理加以介绍并且对算法的代码实现进行详细分析，旨在加深自己对机器学习算法的理解，熟悉这些算法的分布式实现方式。


### [Spark编程指引（中文翻译）](https://www.gitbook.com/book/endymecy/spark-programming-guide-zh-cn/details)
--
Spark guide chinese — 尹迪


### [Spark源代码解析-酷玩](https://github.com/lw-lin/CoolplaySpark)
--
Coolplay Spark 将包含 Spark 源代码解析、Spark 类库、Spark 代码等。


### [Notes talking about the design and implementation of Apache Spark](http://spark-internals.books.yourtion.com)
--
本文主要讨论 Apache Spark 的设计与实现，重点关注其设计思想、运行原理、实现架构及性能调优，附带讨论与 Hadoop MapReduce 在设计与实现上的区别。不喜欢将该文档称之为“源码分析”，因为本文的主要目的不是去解读实现代码，而是尽量有逻辑地，从设计与实现原理的角度，来理解 job 从产生到执行完成的整个过程，进而去理解整个系统。


### [Spark与Scala学习](http://blog.csdn.net/jasonding1354/article/details/46899317)
--
Spark与Scala学习，内容相对比较多，也相对比较杂。



## Storm

### [《Storm入门》中文版](http://ifeve.com/getting-started-with-stom-index/)
--
本文翻译自《[Getting Started With Storm](http://ifeve.com/wp-content/uploads/2014/03/Getting-Started-With-Storm-Jonathan-Leibiusky-Gabriel-E_1276.pdf)》译者：吴京润    编辑：郭蕾 方腾飞


### [Storm实时计算：流操作入门编程实践](http://shiyanjun.cn/archives/977.html)
--
Storm是一个分布式是实时计算系统，它设计了一种对流和计算的抽象，概念比较简单，实际编程开发起来相对容易。本文实现一个简单的WordCount的例子，及各个组件之间的连接方式。


### [Kafka+Storm+HDFS整合实践](http://shiyanjun.cn/archives/934.html)
--
在基于Hadoop平台的很多应用场景中，我们需要对数据进行离线和实时分析，离线分析可以很容易地借助于Hive来实现统计分析，但是对于实时的需求Hive就不合适了。实时应用场景可以使用Storm，它是一个实时处理系统，它为实时处理类应用提供了一个计算模型，可以很容易地进行编程处理。为了统一离线和实时计算，一般情况下，我们都希望将离线和实时计算的数据源的集合统一起来作为输入，然后将数据的流向分别经由实时系统和离线分析系统，分别进行分析处理，这时我们可以考虑将数据源（如使用Flume收集日志）直接连接一个消息中间件，如Kafka，可以整合Flume+Kafka，Flume作为消息的Producer，生产的消息数据（日志数据、业务请求数据等等）发布到Kafka中，然后通过订阅的方式，使用Storm的Topology作为消息的Consumer，在Storm集群中分别进行处理。


### [Storm归档目录-并发编程网](http://ifeve.com/category/storm/)
--
并发编程网的Storm相关文档。


### [storm-kafka源码走读](http://blog.csdn.net/wzhg0508/article/details/40869937)
--
对Storm-kafka module的源码做一点简单的介绍。
