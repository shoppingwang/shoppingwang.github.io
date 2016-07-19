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

> 汇集网络各种资源，包括但不限于框架、类库、软件等等。

## Git

### [闯过这 54 关，点亮你的 Git 技能树](https://www.v2ex.com/t/247270)
如今， Git 大行其道，颇有一统天下之势。
如果你的技能树上 Git 和 Github 的图标还没有点亮的话，你都不好意思说你是程序员。
别说互联网企业，我接触到的许多传统企业都在从 SVN ， Clear Case 等迁移到 Git 上，甚至大厂还会有一个团队去定制适合自己企业的 Git 服务器。

很多人简历上写的「精通 Git 与 Github 」，但如果你问他熟悉到什么程度的话，回答通常是「就是会用常用的 add，commit，push 操作」。


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


## Spark
### [Kafka-Spark-Consumer](https://github.com/dibbhatt/kafka-spark-consumer)
--
High Performance Kafka Consumer for Spark Streaming.


### [Spark机器学习算法研究和源码分析](https://github.com/endymecy/spark-ml-source-analysis)
本项目对spark ml包中各种算法的原理加以介绍并且对算法的代码实现进行详细分析，旨在加深自己对机器学习算法的理解，熟悉这些算法的分布式实现方式。
