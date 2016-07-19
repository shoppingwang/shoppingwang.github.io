---
layout:     post
title:      "Spark Configuration Translation"
subtitle:   "for latest"
date:       2016-07-17
author:     "xp"
header-img: "img/post-bg-universe.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Spark
    - 译文
---
* This will become a table of contents (this text will be scraped).
{:toc}

Spark有三个地方来配置整个系统：

* [Spark properties](#spark-properties) 可以配置大多数的系统参数，通过使用[SparkConf](api/scala/index.html#org.apache.spark.SparkConf)对象进行配置，或者使用Java系统参数进行配置。
* [Environment variables](#environment-variables) 可以对每个机器进行配置，例如通过 `conf/spark-env.sh`脚本文件可配置每个机器的IP。
* [Logging](#configuring-logging) 可以通过 `log4j.properties`文件进行配置。

# Spark Properties

Spark properties控制着大多数据的应用参数设置，并且能对每个应用进行单独设置。这些参数能在[SparkConf](api/scala/index.html#org.apache.spark.SparkConf)直接进行设置然后被应用至`SparkContext`。`SparkConf`允许你设置一些通用的属性（比如master URL和应用名称），也可以通过`set()`方法设置任意的键值对参数。比如，我们可以像下面这样给应用程序初始化两个线程：

我们使用local[2]来运行应用，这意味着有两个线程 - 代表了“最小的”并行化运行条件，它能帮我们检测出只有在分布式环境下才能出现的BUG。

{% highlight scala %}
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
{% endhighlight %}

在本地模式中，我们能使用多于1个的线程数据，比如在Spark Streaming应用中，实际上我们至少需要1个以上的线程来防止任何的线程饥饿问题。

一些需要指定持续时间的设置，可以使用相应的时间单位。
可以使用下面的这些格式：

    25ms (milliseconds)
    5s (seconds)
    10m or 10min (minutes)
    3h (hours)
    5d (days)
    1y (years)


一些需要指定文件字节大小的设置，可以使用相应的文件大小的单位。
可以使用下面的这些格式：

    1b (bytes)
    1k or 1kb (kibibytes = 1024 bytes)
    1m or 1mb (mebibytes = 1024 kibibytes)
    1g or 1gb (gibibytes = 1024 mebibytes)
    1t or 1tb (tebibytes = 1024 gibibytes)
    1p or 1pb (pebibytes = 1024 tebibytes)

## Dynamically Loading Spark Properties
在一些应用场景中，你想避免在`SparkConf`对某些参数进行硬编码。举个例子，你想使用不同的master或者不同的内存设置来跑同一个应用。Spark允许你简单的创建一个空的conf对象：

{% highlight scala %}
val sc = new SparkContext(new SparkConf())
{% endhighlight %}

然后，你就可以在运行时对参数的值进行设置：
{% highlight bash %}
./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
{% endhighlight %}

Spark shell和[`spark-submit`](submitting-applications.html)工具有两种方式来支持配置的动态加载。第一种方式是通过命令行对参数进行设置，例如上面展示的`--master`参数。`spark-submit`使用`--conf`标记能够接受任意的Spark属性，但是为指定Spark属性使用特殊的标记将会成为启动Spark应用时的一部分。运行`./bin/spark-submit --help`命令将可以完整的展示出这些可选项参数。

`bin/spark-submit`也会从`conf/spark-defaults.conf`文件中读取配置信息，文件中的每一行由空白字符分隔的键值对组成，比如：

    spark.master            spark://5.6.7.8:7077
    spark.executor.memory   4g
    spark.eventLog.enabled  true
    spark.serializer        org.apache.spark.serializer.KryoSerializer

任何在命令行或者属性文件指定的参数都会传入到应用中，并且会在SparkConf对象中对参数进行合并。直接在SparkConf中设置的参数具有最高的优先级，其次是通过`spark-submit`和`spark-shell`设置的命令行参数，最后是`spark-defaults.conf`文件中的参数。在早期的Spark版本中，有一些配置的键已经被重命名了，在这种情况下，老的键的名称依然是可用的，只是它的优先级会低于它被命名后的新键。

## Viewing Spark Properties

在`http://<driver>:4040` 页面的"Environment"标签页中列出了Spark应用中的属性配置。在这里，你可以检查这些属性以确认你真的配置对了应用的属性。注意，只有显示的从`spark-defaults.conf`、`SparkConf` 、命令行中加载的参数才会被展示，对于其他的一些配置参数，你可以假定它使用的是默认参数值。

## Available Properties

大多数的控制内部行为的参数已经设置了合理的默认值。一些通用的配置项如下所示：

#### Application Properties
<table>
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.app.name</code></td>
  <td>(none)</td>
  <td>
    应用的名称，将会出现在UI和日志文件中。
  </td>
</tr>
<tr>
  <td><code>spark.driver.cores</code></td>
  <td>1</td>
  <td>
    Driver进程使用的cpu个数，只在cluster模式中生效。
  </td>
</tr>
  <td><code>spark.driver.maxResultSize</code></td>
  <td>1g</td>
  <td>
    限定每个Spark action操作所能返回的最大的序列化结果集大小（例如collect操作）。至少设置为1M，或者设置为0表示无限制。如果返回数据集的大小超过此限制值，任务将忽略其余的数据。设置一个较高的限制可能会引发out-of-memory错误（依赖于spark.driver.memory的设置和JVM的对象内存限制）。正确设置此值能避免driver出现out-of-memory错误。
  </td>
</tr>
<tr>
  <td><code>spark.driver.memory</code></td>
  <td>1g</td>
  <td>
    Driver进程能使用的内存总量，比如，在SparkContext初始化完成后
    (e.g. <code>1g</code>, <code>2g</code>)。

    <br /><em>注意：</em>在client模式中，这个配置项不能通过<code>SparkConf</code>对象进行直接设置，因为driver的JVM在那个时候已经启动了。换言之，你可以通过<code>--driver-memory</code>命令行选项进行设置，或者使用默认的属性配置文件。
  </td>
</tr>
<tr>
  <td><code>spark.executor.memory</code></td>
  <td>1g</td>
  <td>
    每个executor进程可以使用的内存总量(e.g. <code>2g</code>, <code>8g</code>)。
  </td>
</tr>
<tr>
  <td><code>spark.extraListeners</code></td>
  <td>(none)</td>
  <td>
    一个实现了<code>SparkListener</code>接口的以逗号分隔的类名称列表。当初始化SparkContext时，这些类会被实例化并被注册至Spark监听总线中。如果监听实现类中带有单个SparkConf参数的构造器，那么此构造器将会被调用，否则默认无参的构造器将会被调用。如果没有合法的构造器可以被使用，那么SparkContext的创建将会失败并且会抛出一个异常。
  </td>
</tr>
<tr>
  <td><code>spark.local.dir</code></td>
  <td>/tmp</td>
  <td>
    在Spark中被"scratch"空间所使用的目录，包含了map操作的输出文件和保存在硬盘上的RDD。这个目录应该是你的系统中一个快速的本地磁盘。它也可以是一个由逗号分隔的不同磁盘上的不同目录列表。

    注意：在Spark 1.0和以后的版本中，这个参数被SPARK_LOCAL_DIRS (Standalone, Mesos)或被集群管理器设置的环境变量LOCAL_DIRS (YARN)所覆盖。
  </td>
</tr>
<tr>
  <td><code>spark.logConf</code></td>
  <td>false</td>
  <td>
    当SparkContext启动后，以INFO日志级别打印生效的SparkConf信息。
  </td>
</tr>
<tr>
  <td><code>spark.master</code></td>
  <td>(none)</td>
  <td>
    集群管理器的连接地址。参见<a href="submitting-applications.html#master-urls">master允许的地址</a>。
  </td>
</tr>
<tr>
  <td><code>spark.submit.deployMode</code></td>
  <td>(none)</td>
  <td>
    Spark driver的部署模式，可以为"client"或者"cluster"模式，意思就是以本地模式（"cliet"）或者在集群（"cluster"）的某一个远程节点中启动dirver程序。
  </td>
</tr>
</table>

除了上面这些参数，下面的这些参数也是可用的，并且在某些场景下可能是有用的：

#### Runtime Environment
<table>
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.driver.extraClassPath</code></td>
  <td>(none)</td>
  <td>
    被额外附加在driver上的classpath参数。

    <br /><em>注意：</em>在客户端模式中，不能在应用的<code>SparkConf</code>中直接设置此变量，因为此时diver的JVM已经启动了。换句话说，请使用<code>--driver-class-path</code>命令行的方式或者使用默认的属性文件来设置此值。
    </td>
  </td>
</tr>
<tr>
  <td><code>spark.driver.extraJavaOptions</code></td>
  <td>(none)</td>
  <td>
    被额外传递给dirver的JVM选项字符串。例如，GC的设置或者其他的日志设置。注意，用这种方式设置最大的堆大小(-Xmx)是非法的。最大的堆大小设置，在cluster模式中可以通过<code>--driver-memory</code>命令行的选项进行设置。

    <br /><em>注意：</em>在客户端模式中，不能在应用的<code>SparkConf</code>中直接设置此变量，因为此时diver的JVM已经启动了。换句话说，请使用<code>--driver-java-options</code>命令行的方式或者使用默认的属性文件来设置此值。
  </td>
</tr>
<tr>
  <td><code>spark.driver.extraLibraryPath</code></td>
  <td>(none)</td>
  <td>
    当dirver启动时，为JVM设置一个特殊的库路径。

    <br /><em>注意：</em>在客户端模式中，不能在应用的<code>SparkConf</code>中直接设置此变量，因为此时diver的JVM已经启动了。换句话说，请使用<code>--driver-library-path</code>命令行的方式或者使用默认的属性文件来设置此值。
  </td>
</tr>
<tr>
  <td><code>spark.driver.userClassPathFirst</code></td>
  <td>false</td>
  <td>
    (Experimental) 当在driver中加载类时，是否让用户给定的jar文件优先于Spark自己的jar文件加载。这个特性可以用于规避Spark的依赖和用户依赖之间的冲突问题。这个特性现在只是一个实验性质的。
    
    只能被应用于cluster模式中。
  </td>
</tr>
<tr>
  <td><code>spark.executor.extraClassPath</code></td>
  <td>(none)</td>
  <td>
    额外附加给executor的classpath参数。现存的主要的参数是和Spark的旧版本向后兼容的。用户基本上不需要去设置这些参数。
  </td>
</tr>
<tr>
  <td><code>spark.executor.extraJavaOptions</code></td>
  <td>(none)</td>
  <td>
    被额外传递给executors的JVM选项字符串。例如，GC的设置或者其他的日志设置。注意，用这种方式设置最大的堆大小(-Xmx)是非法的。Spark的属性应该使用SparkConf对象或者在使用spark-submit脚本时使用spark-defaults.conf文件进行设置。最大的堆大小设置可以使用spark.executor.memory选项进行设置。
  </td>
</tr>
<tr>
  <td><code>spark.executor.extraLibraryPath</code></td>
  <td>(none)</td>
  <td>
    当executor启动时，为JVM设置一个特殊的库路径。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.maxRetainedFiles</code></td>
  <td>(none)</td>
  <td>
    设置系统保留最新滚动日志文件的个数。
    旧的日志文件会被删除。默认此功能是禁用的。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.maxSize</code></td>
  <td>(none)</td>
  <td>
    设置executor日志文件的最大大小。
    日志滚动替换默认是禁用的。
    参见<code>spark.executor.logs.rolling.maxRetainedFiles</code>参数开启自动清理旧日志。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.strategy</code></td>
  <td>(none)</td>
  <td>
    设置executor日志文件的滚动策略。此项设置默认是禁用的，可以被设置为"time"（基于时间的日志滚动）或者"size"（基本文件大小的滚动）。
    如果是"time" ，使用<code>spark.executor.logs.rolling.time.interval</code>参数设置滚动间隔。
    如果是"size"，使用<code>spark.executor.logs.rolling.maxSize</code>参数设置滚动文件最大大小。
  </td>
</tr>
<tr>
  <td><code>spark.executor.logs.rolling.time.interval</code></td>
  <td>daily</td>
  <td>
    设置executor日志文件轮换的时间间隔。
    日志轮换默认是禁用的。可选的参数值有<code>daily</code>、<code>hourly</code>、<code>minutely</code>或者任何的以秒级时间间隔。参见<code>spark.executor.logs.rolling.maxRetainedFiles</code>开启自动清理旧日志。
  </td>
</tr>
<tr>
  <td><code>spark.executor.userClassPathFirst</code></td>
  <td>false</td>
  <td>
    (Experimental) 和<code>spark.driver.userClassPathFirst</code>参数的功能一样，但是只是应用在executor实例上而已。
  </td>
</tr>
<tr>
  <td><code>spark.executorEnv.[EnvironmentVariableName]</code></td>
  <td>(none)</td>
  <td>
    通过指定<code>EnvironmentVariableName</code>参数给Executor进程增加环境变量。用户可以多个这样的参数从而设置多个这样的环境变量。
  </td>
</tr>
<tr>
  <td><code>spark.python.profile</code></td>
  <td>false</td>
  <td>
    Enable profiling in Python worker, the profile result will show up by <code>sc.show_profiles()</code>,
    or it will be displayed before the driver exiting. It also can be dumped into disk by
    <code>sc.dump_profiles(path)</code>. If some of the profile results had been displayed manually,
    they will not be displayed automatically before driver exiting.

    By default the <code>pyspark.profiler.BasicProfiler</code> will be used, but this can be overridden by
    passing a profiler class in as a parameter to the <code>SparkContext</code> constructor.
  </td>
</tr>
<tr>
  <td><code>spark.python.profile.dump</code></td>
  <td>(none)</td>
  <td>
    The directory which is used to dump the profile result before driver exiting.
    The results will be dumped as separated file for each RDD. They can be loaded
    by ptats.Stats(). If this is specified, the profile result will not be displayed
    automatically.
  </td>
</tr>
<tr>
  <td><code>spark.python.worker.memory</code></td>
  <td>512m</td>
  <td>
    Amount of memory to use per python worker process during aggregation, in the same
    format as JVM memory strings (e.g. <code>512m</code>, <code>2g</code>). If the memory
    used during aggregation goes above this amount, it will spill the data into disks.
  </td>
</tr>
<tr>
  <td><code>spark.python.worker.reuse</code></td>
  <td>true</td>
  <td>
    Reuse Python worker or not. If yes, it will use a fixed number of Python workers,
    does not need to fork() a Python process for every tasks. It will be very useful
    if there is large broadcast, then the broadcast will not be needed to transferred
    from JVM to Python worker for every task.
  </td>
</tr>
<tr>
  <td><code>spark.files</code></td>
  <td></td>
  <td>
    以逗号分隔的传递给每个executor工作目录的文件列表。
  </td>
</tr>
<tr>
  <td><code>spark.submit.pyFiles</code></td>
  <td></td>
  <td>
    Comma-separated list of .zip, .egg, or .py files to place on the PYTHONPATH for Python apps.
  </td>
</tr>
<tr>
  <td><code>spark.jars</code></td>
  <td></td>
  <td>
    以逗号分隔的传递给driver和executor的本地jar文件列表。
  </td>
</tr>
<tr>
  <td><code>spark.jars.packages</code></td>
  <td></td>
  <td>
    以逗号分隔的传递给driver和executor类路径的maven管理的依赖列表。会依次搜索本地maven仓库、maven中心仓库、任何在<code>spark.jars.ivy</code>中设置的远程仓库。依赖的格式是groupId:artifactId:version。
  </td>
</tr>
<tr>
  <td><code>spark.jars.excludes</code></td>
  <td></td>
  <td>
    以逗号分隔的groupId:artifactId格式的列表，用来排除在<code>spark.jars.packages</code>中指定的不需要的依赖，避免Jar冲突。
  </td>
</tr>
<tr>
  <td><code>spark.jars.ivy</code></td>
  <td></td>
  <td>
    为搜索在<code>spark.jars.packages</code>中指定的依赖，以逗号分隔增加需要搜索的远程仓库列表。
  </td>
</tr>
</table>

#### Shuffle Behavior
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.reducer.maxSizeInFlight</code></td>
  <td>48m</td>
  <td>
    Maximum size of map outputs to fetch simultaneously from each reduce task. Since
    each output requires us to create a buffer to receive it, this represents a fixed memory
    overhead per reduce task, so keep it small unless you have a large amount of memory.
  </td>
</tr>
<tr>
  <td><code>spark.reducer.maxReqsInFlight</code></td>
  <td>Int.MaxValue</td>
  <td>
    This configuration limits the number of remote requests to fetch blocks at any given point.
    When the number of hosts in the cluster increase, it might lead to very large number
    of in-bound connections to one or more nodes, causing the workers to fail under load.
    By allowing it to limit the number of fetch requests, this scenario can be mitigated.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.compress</code></td>
  <td>true</td>
  <td>
    Whether to compress map output files. Generally a good idea. Compression will use
    <code>spark.io.compression.codec</code>.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.file.buffer</code></td>
  <td>32k</td>
  <td>
    Size of the in-memory buffer for each shuffle file output stream. These buffers
    reduce the number of disk seeks and system calls made in creating intermediate shuffle files.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.maxRetries</code></td>
  <td>3</td>
  <td>
    (Netty only) Fetches that fail due to IO-related exceptions are automatically retried if this is
    set to a non-zero value. This retry logic helps stabilize large shuffles in the face of long GC
    pauses or transient network connectivity issues.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.numConnectionsPerPeer</code></td>
  <td>1</td>
  <td>
    (Netty only) Connections between hosts are reused in order to reduce connection buildup for
    large clusters. For clusters with many hard disks and few hosts, this may result in insufficient
    concurrency to saturate all disks, and so users may consider increasing this value.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.preferDirectBufs</code></td>
  <td>true</td>
  <td>
    (Netty only) Off-heap buffers are used to reduce garbage collection during shuffle and cache
    block transfer. For environments where off-heap memory is tightly limited, users may wish to
    turn this off to force all allocations from Netty to be on-heap.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.retryWait</code></td>
  <td>5s</td>
  <td>
    (Netty only) How long to wait between retries of fetches. The maximum delay caused by retrying
    is 15 seconds by default, calculated as <code>maxRetries * retryWait</code>.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.service.enabled</code></td>
  <td>false</td>
  <td>
    Enables the external shuffle service. This service preserves the shuffle files written by
    executors so the executors can be safely removed. This must be enabled if
    <code>spark.dynamicAllocation.enabled</code> is "true". The external shuffle service
    must be set up in order to enable it. See
    <a href="job-scheduling.html#configuration-and-setup">dynamic allocation
    configuration and setup documentation</a> for more information.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.service.port</code></td>
  <td>7337</td>
  <td>
    Port on which the external shuffle service will run.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.sort.bypassMergeThreshold</code></td>
  <td>200</td>
  <td>
    (Advanced) In the sort-based shuffle manager, avoid merge-sorting data if there is no
    map-side aggregation and there are at most this many reduce partitions.
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.spill.compress</code></td>
  <td>true</td>
  <td>
    Whether to compress data spilled during shuffles. Compression will use
    <code>spark.io.compression.codec</code>.
  </td>
</tr>
</table>

#### Spark UI
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.eventLog.compress</code></td>
  <td>false</td>
  <td>
    Whether to compress logged events, if <code>spark.eventLog.enabled</code> is true.
  </td>
</tr>
<tr>
  <td><code>spark.eventLog.dir</code></td>
  <td>file:///tmp/spark-events</td>
  <td>
    Base directory in which Spark events are logged, if <code>spark.eventLog.enabled</code> is true.
    Within this base directory, Spark creates a sub-directory for each application, and logs the
    events specific to the application in this directory. Users may want to set this to
    a unified location like an HDFS directory so history files can be read by the history server.
  </td>
</tr>
<tr>
  <td><code>spark.eventLog.enabled</code></td>
  <td>false</td>
  <td>
    Whether to log Spark events, useful for reconstructing the Web UI after the application has
    finished.
  </td>
</tr>
<tr>
  <td><code>spark.ui.killEnabled</code></td>
  <td>true</td>
  <td>
    Allows stages and corresponding jobs to be killed from the web ui.
  </td>
</tr>
<tr>
  <td><code>spark.ui.port</code></td>
  <td>4040</td>
  <td>
    Port for your application's dashboard, which shows memory and workload data.
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedJobs</code></td>
  <td>1000</td>
  <td>
    How many jobs the Spark UI and status APIs remember before garbage
    collecting.
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedStages</code></td>
  <td>1000</td>
  <td>
    How many stages the Spark UI and status APIs remember before garbage
    collecting.
  </td>
</tr>
<tr>
  <td><code>spark.worker.ui.retainedExecutors</code></td>
  <td>1000</td>
  <td>
    How many finished executors the Spark UI and status APIs remember before garbage collecting.
  </td>
</tr>
<tr>
  <td><code>spark.worker.ui.retainedDrivers</code></td>
  <td>1000</td>
  <td>
    How many finished drivers the Spark UI and status APIs remember before garbage collecting.
  </td>
</tr>
<tr>
  <td><code>spark.sql.ui.retainedExecutions</code></td>
  <td>1000</td>
  <td>
    How many finished executions the Spark UI and status APIs remember before garbage collecting.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.ui.retainedBatches</code></td>
  <td>1000</td>
  <td>
    How many finished batches the Spark UI and status APIs remember before garbage collecting.
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedDeadExecutors</code></td>
  <td>100</td>
  <td>
    How many dead executors the Spark UI and status APIs remember before garbage collecting.
  </td>
</tr>
</table>

#### Compression and Serialization
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.broadcast.compress</code></td>
  <td>true</td>
  <td>
    Whether to compress broadcast variables before sending them. Generally a good idea.
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.codec</code></td>
  <td>lz4</td>
  <td>
    The codec used to compress internal data such as RDD partitions, broadcast variables and
    shuffle outputs. By default, Spark provides three codecs: <code>lz4</code>, <code>lzf</code>,
    and <code>snappy</code>. You can also use fully qualified class names to specify the codec,
    e.g.
    <code>org.apache.spark.io.LZ4CompressionCodec</code>,
    <code>org.apache.spark.io.LZFCompressionCodec</code>,
    and <code>org.apache.spark.io.SnappyCompressionCodec</code>.
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.lz4.blockSize</code></td>
  <td>32k</td>
  <td>
    Block size used in LZ4 compression, in the case when LZ4 compression codec
    is used. Lowering this block size will also lower shuffle memory usage when LZ4 is used.
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.snappy.blockSize</code></td>
  <td>32k</td>
  <td>
    Block size used in Snappy compression, in the case when Snappy compression codec
    is used. Lowering this block size will also lower shuffle memory usage when Snappy is used.
  </td>
</tr>
<tr>
  <td><code>spark.kryo.classesToRegister</code></td>
  <td>(none)</td>
  <td>
    If you use Kryo serialization, give a comma-separated list of custom class names to register
    with Kryo.
    See the <a href="tuning.html#data-serialization">tuning guide</a> for more details.
  </td>
</tr>
<tr>
  <td><code>spark.kryo.referenceTracking</code></td>
  <td>true (false when using Spark SQL Thrift Server)</td>
  <td>
    Whether to track references to the same object when serializing data with Kryo, which is
    necessary if your object graphs have loops and useful for efficiency if they contain multiple
    copies of the same object. Can be disabled to improve performance if you know this is not the
    case.
  </td>
</tr>
<tr>
  <td><code>spark.kryo.registrationRequired</code></td>
  <td>false</td>
  <td>
    Whether to require registration with Kryo. If set to 'true', Kryo will throw an exception
    if an unregistered class is serialized. If set to false (the default), Kryo will write
    unregistered class names along with each object. Writing class names can cause
    significant performance overhead, so enabling this option can enforce strictly that a
    user has not omitted classes from registration.
  </td>
</tr>
<tr>
  <td><code>spark.kryo.registrator</code></td>
  <td>(none)</td>
  <td>
    If you use Kryo serialization, give a comma-separated list of classes that register your custom classes with Kryo. This
    property is useful if you need to register your classes in a custom way, e.g. to specify a custom
    field serializer. Otherwise <code>spark.kryo.classesToRegister</code> is simpler. It should be
    set to classes that extend
    <a href="api/scala/index.html#org.apache.spark.serializer.KryoRegistrator">
    <code>KryoRegistrator</code></a>.
    See the <a href="tuning.html#data-serialization">tuning guide</a> for more details.
  </td>
</tr>
<tr>
  <td><code>spark.kryoserializer.buffer.max</code></td>
  <td>64m</td>
  <td>
    Maximum allowable size of Kryo serialization buffer. This must be larger than any
    object you attempt to serialize. Increase this if you get a "buffer limit exceeded" exception
    inside Kryo.
  </td>
</tr>
<tr>
  <td><code>spark.kryoserializer.buffer</code></td>
  <td>64k</td>
  <td>
    Initial size of Kryo's serialization buffer. Note that there will be one buffer
     <i>per core</i> on each worker. This buffer will grow up to
     <code>spark.kryoserializer.buffer.max</code> if needed.
  </td>
</tr>
<tr>
  <td><code>spark.rdd.compress</code></td>
  <td>false</td>
  <td>
    Whether to compress serialized RDD partitions (e.g. for
    <code>StorageLevel.MEMORY_ONLY_SER</code> in Java
    and Scala or <code>StorageLevel.MEMORY_ONLY</code> in Python).
    Can save substantial space at the cost of some extra CPU time.
  </td>
</tr>
<tr>
  <td><code>spark.serializer</code></td>
  <td>
    指定用来序列化的类库，包括通过网络传输数据或缓存数据时的序列化。默认的Java序列化对于任何可以被序列化的Java对象都适用，但是速度很慢。我们推荐在追求速度时使用org.apache.spark.serializer.KryoSerializer并对Kryo进行适当的调优。该项可以配置为任何org.apache.spark.Serializer的子类。
  </td>
  <td>
    Class to use for serializing objects that will be sent over the network or need to be cached
    in serialized form. The default of Java serialization works with any Serializable Java object
    but is quite slow, so we recommend <a href="tuning.html">using
    <code>org.apache.spark.serializer.KryoSerializer</code> and configuring Kryo serialization</a>
    when speed is necessary. Can be any subclass of
    <a href="api/scala/index.html#org.apache.spark.serializer.Serializer">
    <code>org.apache.spark.Serializer</code></a>.
  </td>
</tr>
<tr>
  <td><code>spark.serializer.objectStreamReset</code></td>
  <td>100</td>
  <td>
    When serializing using org.apache.spark.serializer.JavaSerializer, the serializer caches
    objects to prevent writing redundant data, however that stops garbage collection of those
    objects. By calling 'reset' you flush that info from the serializer, and allow old
    objects to be collected. To turn off this periodic reset set it to -1.
    By default it will reset the serializer every 100 objects.
  </td>
</tr>
</table>

#### Memory Management
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.memory.fraction</code></td>
  <td>0.6</td>
  <td>
    Fraction of (heap space - 300MB) used for execution and storage. The lower this is, the
    more frequently spills and cached data eviction occur. The purpose of this config is to set
    aside memory for internal metadata, user data structures, and imprecise size estimation
    in the case of sparse, unusually large records. Leaving this at the default value is
    recommended. For more detail, including important information about correctly tuning JVM
    garbage collection when increasing this value, see
    <a href="tuning.html#memory-management-overview">this description</a>.
  </td>
</tr>
<tr>
  <td><code>spark.memory.storageFraction</code></td>
  <td>0.5</td>
  <td>
    Amount of storage memory immune to eviction, expressed as a fraction of the size of the
    region set aside by <code>s​park.memory.fraction</code>. The higher this is, the less
    working memory may be available to execution and tasks may spill to disk more often.
    Leaving this at the default value is recommended. For more detail, see
    <a href="tuning.html#memory-management-overview">this description</a>.
  </td>
</tr>
<tr>
  <td><code>spark.memory.offHeap.enabled</code></td>
  <td>false</td>
  <td>
    If true, Spark will attempt to use off-heap memory for certain operations. If off-heap memory use is enabled, then <code>spark.memory.offHeap.size</code> must be positive.
  </td>
</tr>
<tr>
  <td><code>spark.memory.offHeap.size</code></td>
  <td>0</td>
  <td>
    The absolute amount of memory in bytes which can be used for off-heap allocation.
    This setting has no impact on heap memory usage, so if your executors' total memory consumption must fit within some hard limit then be sure to shrink your JVM heap size accordingly.
    This must be set to a positive value when <code>spark.memory.offHeap.enabled=true</code>.
  </td>
</tr>
<tr>
  <td><code>spark.memory.useLegacyMode</code></td>
  <td>false</td>
  <td>
    ​Whether to enable the legacy memory management mode used in Spark 1.5 and before.
    The legacy mode rigidly partitions the heap space into fixed-size regions,
    potentially leading to excessive spilling if the application was not tuned.
    The following deprecated memory fraction configurations are not read unless this is enabled:
    <code>spark.shuffle.memoryFraction</code><br>
    <code>spark.storage.memoryFraction</code><br>
    <code>spark.storage.unrollFraction</code>
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.memoryFraction</code></td>
  <td>0.2</td>
  <td>
    (deprecated) This is read only if <code>spark.memory.useLegacyMode</code> is enabled.
    Fraction of Java heap to use for aggregation and cogroups during shuffles.
    At any given time, the collective size of
    all in-memory maps used for shuffles is bounded by this limit, beyond which the contents will
    begin to spill to disk. If spills are often, consider increasing this value at the expense of
    <code>spark.storage.memoryFraction</code>.
  </td>
</tr>
<tr>
  <td><code>spark.storage.memoryFraction</code></td>
  <td>0.6</td>
  <td>
    (deprecated) This is read only if <code>spark.memory.useLegacyMode</code> is enabled.
    Fraction of Java heap to use for Spark's memory cache. This should not be larger than the "old"
    generation of objects in the JVM, which by default is given 0.6 of the heap, but you can
    increase it if you configure your own old generation size.
  </td>
</tr>
<tr>
  <td><code>spark.storage.unrollFraction</code></td>
  <td>0.2</td>
  <td>
    (deprecated) This is read only if <code>spark.memory.useLegacyMode</code> is enabled.
    Fraction of <code>spark.storage.memoryFraction</code> to use for unrolling blocks in memory.
    This is dynamically allocated by dropping existing blocks when there is not enough free
    storage space to unroll the new block in its entirety.
  </td>
</tr>
</table>

#### Execution Behavior
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.broadcast.blockSize</code></td>
  <td>4m</td>
  <td>
    Size of each piece of a block for <code>TorrentBroadcastFactory</code>.
    Too large a value decreases parallelism during broadcast (makes it slower); however, if it is
    too small, <code>BlockManager</code> might take a performance hit.
  </td>
</tr>
<tr>
  <td><code>spark.executor.cores</code></td>
  <td>
    1 in YARN mode, all the available cores on the worker in
    standalone and Mesos coarse-grained modes.
  </td>
  <td>
    The number of cores to use on each executor.

    In standalone and Mesos coarse-grained modes, setting this
    parameter allows an application to run multiple executors on the
    same worker, provided that there are enough cores on that
    worker. Otherwise, only one executor per application will run on
    each worker.
  </td>
</tr>
<tr>
  <td><code>spark.default.parallelism</code></td>
  <td>
    For distributed shuffle operations like <code>reduceByKey</code> and <code>join</code>, the
    largest number of partitions in a parent RDD.  For operations like <code>parallelize</code>
    with no parent RDDs, it depends on the cluster manager:
    <ul>
      <li>Local mode: number of cores on the local machine</li>
      <li>Mesos fine grained mode: 8</li>
      <li>Others: total number of cores on all executor nodes or 2, whichever is larger</li>
    </ul>
  </td>
  <td>
    Default number of partitions in RDDs returned by transformations like <code>join</code>,
    <code>reduceByKey</code>, and <code>parallelize</code> when not set by user.
  </td>
</tr>
<tr>
    <td><code>spark.executor.heartbeatInterval</code></td>
    <td>10s</td>
    <td>Interval between each executor's heartbeats to the driver.  Heartbeats let
    the driver know that the executor is still alive and update it with metrics for in-progress
    tasks.</td>
</tr>
<tr>
  <td><code>spark.files.fetchTimeout</code></td>
  <td>60s</td>
  <td>
    Communication timeout to use when fetching files added through SparkContext.addFile() from
    the driver.
  </td>
</tr>
<tr>
  <td><code>spark.files.useFetchCache</code></td>
  <td>true</td>
  <td>
    If set to true (default), file fetching will use a local cache that is shared by executors
    that belong to the same application, which can improve task launching performance when
    running many executors on the same host. If set to false, these caching optimizations will
    be disabled and all executors will fetch their own copies of files. This optimization may be
    disabled in order to use Spark local directories that reside on NFS filesystems (see
    <a href="https://issues.apache.org/jira/browse/SPARK-6313">SPARK-6313</a> for more details).
  </td>
</tr>
<tr>
  <td><code>spark.files.overwrite</code></td>
  <td>false</td>
  <td>
    Whether to overwrite files added through SparkContext.addFile() when the target file exists and
    its contents do not match those of the source.
  </td>
</tr>
<tr>
    <td><code>spark.hadoop.cloneConf</code></td>
    <td>false</td>
    <td>If set to true, clones a new Hadoop <code>Configuration</code> object for each task.  This
    option should be enabled to work around <code>Configuration</code> thread-safety issues (see
    <a href="https://issues.apache.org/jira/browse/SPARK-2546">SPARK-2546</a> for more details).
    This is disabled by default in order to avoid unexpected performance regressions for jobs that
    are not affected by these issues.</td>
</tr>
<tr>
    <td><code>spark.hadoop.validateOutputSpecs</code></td>
    <td>true</td>
    <td>If set to true, validates the output specification (e.g. checking if the output directory already exists)
    used in saveAsHadoopFile and other variants. This can be disabled to silence exceptions due to pre-existing
    output directories. We recommend that users do not disable this except if trying to achieve compatibility with
    previous versions of Spark. Simply use Hadoop's FileSystem API to delete output directories by hand.
    This setting is ignored for jobs generated through Spark Streaming's StreamingContext, since
    data may need to be rewritten to pre-existing output directories during checkpoint recovery.</td>
</tr>
<tr>
  <td><code>spark.storage.memoryMapThreshold</code></td>
  <td>2m</td>
  <td>
    Size of a block above which Spark memory maps when reading a block from disk.
    This prevents Spark from memory mapping very small blocks. In general, memory
    mapping has high overhead for blocks close to or below the page size of the operating system.
  </td>
</tr>
</table>

#### Networking
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.rpc.message.maxSize</code></td>
  <td>128</td>
  <td>
    Maximum message size (in MB) to allow in "control plane" communication; generally only applies to map
    output size information sent between executors and the driver. Increase this if you are running
    jobs with many thousands of map and reduce tasks and see messages about the RPC message size.
  </td>
</tr>
<tr>
  <td><code>spark.blockManager.port</code></td>
  <td>(random)</td>
  <td>
    Port for all block managers to listen on. These exist on both the driver and the executors.
  </td>
</tr>
<tr>
  <td><code>spark.driver.host</code></td>
  <td>(local hostname)</td>
  <td>
    Hostname or IP address for the driver to listen on.
    This is used for communicating with the executors and the standalone Master.
  </td>
</tr>
<tr>
  <td><code>spark.driver.port</code></td>
  <td>(random)</td>
  <td>
    Port for the driver to listen on.
    This is used for communicating with the executors and the standalone Master.
  </td>
</tr>
<tr>
  <td><code>spark.network.timeout</code></td>
  <td>120s</td>
  <td>
    Default timeout for all network interactions. This config will be used in place of
    <code>spark.core.connection.ack.wait.timeout</code>,
    <code>spark.storage.blockManagerSlaveTimeoutMs</code>,
    <code>spark.shuffle.io.connectionTimeout</code>, <code>spark.rpc.askTimeout</code> or
    <code>spark.rpc.lookupTimeout</code> if they are not configured.
  </td>
</tr>
<tr>
  <td><code>spark.port.maxRetries</code></td>
  <td>16</td>
  <td>
    Maximum number of retries when binding to a port before giving up.
    When a port is given a specific value (non 0), each subsequent retry will
    increment the port used in the previous attempt by 1 before retrying. This
    essentially allows it to try a range of ports from the start port specified
    to port + maxRetries.
  </td>
</tr>
<tr>
  <td><code>spark.rpc.numRetries</code></td>
  <td>3</td>
  <td>
    Number of times to retry before an RPC task gives up.
    An RPC task will run at most times of this number.
  </td>
</tr>
<tr>
  <td><code>spark.rpc.retry.wait</code></td>
  <td>3s</td>
  <td>
    Duration for an RPC ask operation to wait before retrying.
  </td>
</tr>
<tr>
  <td><code>spark.rpc.askTimeout</code></td>
  <td>120s</td>
  <td>
    Duration for an RPC ask operation to wait before timing out.
  </td>
</tr>
<tr>
  <td><code>spark.rpc.lookupTimeout</code></td>
  <td>120s</td>
  <td>
    Duration for an RPC remote endpoint lookup operation to wait before timing out.
  </td>
</tr>
</table>

#### Scheduling
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.cores.max</code></td>
  <td>(not set)</td>
  <td>
    When running on a <a href="spark-standalone.html">standalone deploy cluster</a> or a
    <a href="running-on-mesos.html#mesos-run-modes">Mesos cluster in "coarse-grained"
    sharing mode</a>, the maximum amount of CPU cores to request for the application from
    across the cluster (not from each machine). If not set, the default will be
    <code>spark.deploy.defaultCores</code> on Spark's standalone cluster manager, or
    infinite (all available cores) on Mesos.
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait</code></td>
  <td>3s</td>
  <td>
    How long to wait to launch a data-local task before giving up and launching it
    on a less-local node. The same wait will be used to step through multiple locality levels
    (process-local, node-local, rack-local and then any). It is also possible to customize the
    waiting time for each level by setting <code>spark.locality.wait.node</code>, etc.
    You should increase this setting if your tasks are long and see poor locality, but the
    default usually works well.
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.node</code></td>
  <td>spark.locality.wait</td>
  <td>
    Customize the locality wait for node locality. For example, you can set this to 0 to skip
    node locality and search immediately for rack locality (if your cluster has rack information).
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.process</code></td>
  <td>spark.locality.wait</td>
  <td>
    Customize the locality wait for process locality. This affects tasks that attempt to access
    cached data in a particular executor process.
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.rack</code></td>
  <td>spark.locality.wait</td>
  <td>
    Customize the locality wait for rack locality.
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.maxRegisteredResourcesWaitingTime</code></td>
  <td>30s</td>
  <td>
    Maximum amount of time to wait for resources to register before scheduling begins.
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.minRegisteredResourcesRatio</code></td>
  <td>0.8 for YARN mode; 0.0 for standalone mode and Mesos coarse-grained mode</td>
  <td>
    The minimum ratio of registered resources (registered resources / total expected resources)
    (resources are executors in yarn mode, CPU cores in standalone mode and Mesos coarsed-grained
     mode ['spark.cores.max' value is total expected resources for Mesos coarse-grained mode] )
    to wait for before scheduling begins. Specified as a double between 0.0 and 1.0.
    Regardless of whether the minimum ratio of resources has been reached,
    the maximum amount of time it will wait before scheduling begins is controlled by config
    <code>spark.scheduler.maxRegisteredResourcesWaitingTime</code>.
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.mode</code></td>
  <td>FIFO</td>
  <td>
    The <a href="job-scheduling.html#scheduling-within-an-application">scheduling mode</a> between
    jobs submitted to the same SparkContext. Can be set to <code>FAIR</code>
    to use fair sharing instead of queueing jobs one after another. Useful for
    multi-user services.
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.revive.interval</code></td>
  <td>1s</td>
  <td>
    The interval length for the scheduler to revive the worker resource offers to run tasks.
  </td>
</tr>
<tr>
  <td><code>spark.speculation</code></td>
  <td>false</td>
  <td>
    设为true时开启任务预测执行机制。当出现比较慢的任务时，这种机制会在另外的节点上也尝试执行该任务的一个副本。打开此选项会帮助减少大规模集群中个别较慢的任务带来的影响。
  </td>
</tr>
<tr>
  <td><code>spark.speculation.interval</code></td>
  <td>100ms</td>
  <td>
    How often Spark will check for tasks to speculate.
  </td>
</tr>
<tr>
  <td><code>spark.speculation.multiplier</code></td>
  <td>1.5</td>
  <td>
    How many times slower a task is than the median to be considered for speculation.
  </td>
</tr>
<tr>
  <td><code>spark.speculation.quantile</code></td>
  <td>0.75</td>
  <td>
    Percentage of tasks which must be complete before speculation is enabled for a particular stage.
  </td>
</tr>
<tr>
  <td><code>spark.task.cpus</code></td>
  <td>1</td>
  <td>
    Number of cores to allocate for each task.
  </td>
</tr>
<tr>
  <td><code>spark.task.maxFailures</code></td>
  <td>4</td>
  <td>
    Number of individual task failures before giving up on the job.
    Should be greater than or equal to 1. Number of allowed retries = this value - 1.
  </td>
</tr>
</table>

#### Dynamic Allocation
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.dynamicAllocation.enabled</code></td>
  <td>false</td>
  <td>
    Whether to use dynamic resource allocation, which scales the number of executors registered
    with this application up and down based on the workload. 
    For more detail, see the description
    <a href="job-scheduling.html#dynamic-resource-allocation">here</a>.
    <br><br>
    This requires <code>spark.shuffle.service.enabled</code> to be set.
    The following configurations are also relevant:
    <code>spark.dynamicAllocation.minExecutors</code>,
    <code>spark.dynamicAllocation.maxExecutors</code>, and
    <code>spark.dynamicAllocation.initialExecutors</code>
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.executorIdleTimeout</code></td>
  <td>60s</td>
  <td>
    If dynamic allocation is enabled and an executor has been idle for more than this duration,
    the executor will be removed. For more detail, see this
    <a href="job-scheduling.html#resource-allocation-policy">description</a>.
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.cachedExecutorIdleTimeout</code></td>
  <td>infinity</td>
  <td>
    If dynamic allocation is enabled and an executor which has cached data blocks has been idle for more than this duration,
    the executor will be removed. For more details, see this
    <a href="job-scheduling.html#resource-allocation-policy">description</a>.
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.initialExecutors</code></td>
  <td><code>spark.dynamicAllocation.minExecutors</code></td>
  <td>
    Initial number of executors to run if dynamic allocation is enabled.
    <br /><br />
    If `--num-executors` (or `spark.executor.instances`) is set and larger than this value, it will
    be used as the initial number of executors.
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.maxExecutors</code></td>
  <td>infinity</td>
  <td>
    Upper bound for the number of executors if dynamic allocation is enabled.
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.minExecutors</code></td>
  <td>0</td>
  <td>
    Lower bound for the number of executors if dynamic allocation is enabled.
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.schedulerBacklogTimeout</code></td>
  <td>1s</td>
  <td>
    If dynamic allocation is enabled and there have been pending tasks backlogged for more than
    this duration, new executors will be requested. For more detail, see this
    <a href="job-scheduling.html#resource-allocation-policy">description</a>.
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.sustainedSchedulerBacklogTimeout</code></td>
  <td><code>schedulerBacklogTimeout</code></td>
  <td>
    Same as <code>spark.dynamicAllocation.schedulerBacklogTimeout</code>, but used only for
    subsequent executor requests. For more detail, see this
    <a href="job-scheduling.html#resource-allocation-policy">description</a>.
  </td>
</tr>
</table>

#### Security
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.acls.enable</code></td>
  <td>false</td>
  <td>
    Whether Spark acls should be enabled. If enabled, this checks to see if the user has
    access permissions to view or modify the job.  Note this requires the user to be known,
    so if the user comes across as null no checks are done. Filters can be used with the UI
    to authenticate and set the user.
  </td>
</tr>
<tr>
  <td><code>spark.admin.acls</code></td>
  <td>Empty</td>
  <td>
    Comma separated list of users/administrators that have view and modify access to all Spark jobs.
    This can be used if you run on a shared cluster and have a set of administrators or devs who
    help debug when things do not work. Putting a "*" in the list means any user can have the
    privilege of admin.
  </td>
</tr>
<tr>
  <td><code>spark.admin.acls.groups</code></td>
  <td>Empty</td>
  <td>
    Comma separated list of groups that have view and modify access to all Spark jobs.
    This can be used if you have a set of administrators or developers who help maintain and debug
    the underlying infrastructure. Putting a "*" in the list means any user in any group can have
    the privilege of admin. The user groups are obtained from the instance of the groups mapping
    provider specified by <code>spark.user.groups.mapping</code>. Check the entry
    <code>spark.user.groups.mapping</code> for more details.
  </td>
</tr>
<tr>
  <td><code>spark.user.groups.mapping</code></td>
  <td><code>org.apache.spark.security.ShellBasedGroupsMappingProvider</code></td>
  <td>
    The list of groups for a user are determined by a group mapping service defined by the trait
    org.apache.spark.security.GroupMappingServiceProvider which can configured by this property.
    A default unix shell based implementation is provided <code>org.apache.spark.security.ShellBasedGroupsMappingProvider</code>
    which can be specified to resolve a list of groups for a user.
    <em>Note:</em> This implementation supports only a Unix/Linux based environment. Windows environment is
    currently <b>not</b> supported. However, a new platform/protocol can be supported by implementing
    the trait <code>org.apache.spark.security.GroupMappingServiceProvider</code>.
  </td>
</tr>
<tr>
  <td><code>spark.authenticate</code></td>
  <td>false</td>
  <td>
    Whether Spark authenticates its internal connections. See
    <code>spark.authenticate.secret</code> if not running on YARN.
  </td>
</tr>
<tr>
  <td><code>spark.authenticate.secret</code></td>
  <td>None</td>
  <td>
    Set the secret key used for Spark to authenticate between components. This needs to be set if
    not running on YARN and authentication is enabled.
  </td>
</tr>
<tr>
  <td><code>spark.authenticate.enableSaslEncryption</code></td>
  <td>false</td>
  <td>
    Enable encrypted communication when authentication is enabled. This option is currently
    only supported by the block transfer service.
  </td>
</tr>
<tr>
  <td><code>spark.network.sasl.serverAlwaysEncrypt</code></td>
  <td>false</td>
  <td>
    Disable unencrypted connections for services that support SASL authentication. This is
    currently supported by the external shuffle service.
  </td>
</tr>
<tr>
  <td><code>spark.core.connection.ack.wait.timeout</code></td>
  <td>60s</td>
  <td>
    How long for the connection to wait for ack to occur before timing
    out and giving up. To avoid unwilling timeout caused by long pause like GC,
    you can set larger value.
  </td>
</tr>
<tr>
  <td><code>spark.core.connection.auth.wait.timeout</code></td>
  <td>30s</td>
  <td>
    How long for the connection to wait for authentication to occur before timing
    out and giving up.
  </td>
</tr>
<tr>
  <td><code>spark.modify.acls</code></td>
  <td>Empty</td>
  <td>
    Comma separated list of users that have modify access to the Spark job. By default only the
    user that started the Spark job has access to modify it (kill it for example). Putting a "*" in
    the list means any user can have access to modify it.
  </td>
</tr>
<tr>
  <td><code>spark.modify.acls.groups</code></td>
  <td>Empty</td>
  <td>
    Comma separated list of groups that have modify access to the Spark job. This can be used if you
    have a set of administrators or developers from the same team to have access to control the job.
    Putting a "*" in the list means any user in any group has the access to modify the Spark job.
    The user groups are obtained from the instance of the groups mapping provider specified by
    <code>spark.user.groups.mapping</code>. Check the entry <code>spark.user.groups.mapping</code>
    for more details.
  </td>
</tr>
<tr>
  <td><code>spark.ui.filters</code></td>
  <td>None</td>
  <td>
    Comma separated list of filter class names to apply to the Spark web UI. The filter should be a
    standard <a href="http://docs.oracle.com/javaee/6/api/javax/servlet/Filter.html">
    javax servlet Filter</a>. Parameters to each filter can also be specified by setting a
    java system property of: <br />
    <code>spark.&lt;class name of filter&gt;.params='param1=value1,param2=value2'</code><br />
    For example: <br />
    <code>-Dspark.ui.filters=com.test.filter1</code> <br />
    <code>-Dspark.com.test.filter1.params='param1=foo,param2=testing'</code>
  </td>
</tr>
<tr>
  <td><code>spark.ui.view.acls</code></td>
  <td>Empty</td>
  <td>
    Comma separated list of users that have view access to the Spark web ui. By default only the
    user that started the Spark job has view access. Putting a "*" in the list means any user can
    have view access to this Spark job.
  </td>
</tr>
<tr>
  <td><code>spark.ui.view.acls.groups</code></td>
  <td>Empty</td>
  <td>
    Comma separated list of groups that have view access to the Spark web ui to view the Spark Job
    details. This can be used if you have a set of administrators or developers or users who can
    monitor the Spark job submitted. Putting a "*" in the list means any user in any group can view
    the Spark job details on the Spark web ui. The user groups are obtained from the instance of the
    groups mapping provider specified by <code>spark.user.groups.mapping</code>. Check the entry
    <code>spark.user.groups.mapping</code> for more details.
  </td>
</tr>
</table>

#### Encryption

<table class="table">
    <tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
    <tr>
        <td><code>spark.ssl.enabled</code></td>
        <td>false</td>
        <td>
            <p>Whether to enable SSL connections on all supported protocols.</p>

            <p>All the SSL settings like <code>spark.ssl.xxx</code> where <code>xxx</code> is a
            particular configuration property, denote the global configuration for all the supported
            protocols. In order to override the global configuration for the particular protocol,
            the properties must be overwritten in the protocol-specific namespace.</p>

            <p>Use <code>spark.ssl.YYY.XXX</code> settings to overwrite the global configuration for
            particular protocol denoted by <code>YYY</code>. Currently <code>YYY</code> can be
            only <code>fs</code> for file server.</p>
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.enabledAlgorithms</code></td>
        <td>Empty</td>
        <td>
            A comma separated list of ciphers. The specified ciphers must be supported by JVM.
            The reference list of protocols one can find on
            <a href="https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https">this</a>
            page.
            Note: If not set, it will use the default cipher suites of JVM.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyPassword</code></td>
        <td>None</td>
        <td>
            A password to the private key in key-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStore</code></td>
        <td>None</td>
        <td>
            A path to a key-store file. The path can be absolute or relative to the directory where
            the component is started in.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStorePassword</code></td>
        <td>None</td>
        <td>
            A password to the key-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStoreType</code></td>
        <td>JKS</td>
        <td>
            The type of the key-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.protocol</code></td>
        <td>None</td>
        <td>
            A protocol name. The protocol must be supported by JVM. The reference list of protocols
            one can find on <a href="https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https">this</a>
            page.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.needClientAuth</code></td>
        <td>false</td>
        <td>
            Set true if SSL needs client authentication.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStore</code></td>
        <td>None</td>
        <td>
            A path to a trust-store file. The path can be absolute or relative to the directory
            where the component is started in.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStorePassword</code></td>
        <td>None</td>
        <td>
            A password to the trust-store.
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStoreType</code></td>
        <td>JKS</td>
        <td>
            The type of the trust-store.
        </td>
    </tr>
</table>


#### Spark SQL
Running the <code>SET -v</code> command will show the entire list of the SQL configuration.

<div class="codetabs">
<div data-lang="scala"  markdown="1">

{% highlight scala %}
// spark is an existing SparkSession
spark.sql("SET -v").show(numRows = 200, truncate = false)
{% endhighlight %}

</div>

<div data-lang="java"  markdown="1">

{% highlight java %}
// spark is an existing SparkSession
spark.sql("SET -v").show(200, false);
{% endhighlight %}
</div>

<div data-lang="python"  markdown="1">

{% highlight python %}
# spark is an existing SparkSession
spark.sql("SET -v").show(n=200, truncate=False)
{% endhighlight %}

</div>

<div data-lang="r"  markdown="1">

{% highlight r %}
sparkR.session()
properties <- sql("SET -v")
showDF(properties, numRows = 200, truncate = FALSE)
{% endhighlight %}

</div>
</div>


#### Spark Streaming
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.streaming.backpressure.enabled</code></td>
  <td>false</td>
  <td>
    Enables or disables Spark Streaming's internal backpressure mechanism (since 1.5).
    This enables the Spark Streaming to control the receiving rate based on the
    current batch scheduling delays and processing times so that the system receives
    only as fast as the system can process. Internally, this dynamically sets the
    maximum receiving rate of receivers. This rate is upper bounded by the values
    <code>spark.streaming.receiver.maxRate</code> and <code>spark.streaming.kafka.maxRatePerPartition</code>
    if they are set (see below).
  </td>
</tr>
<tr>
  <td><code>spark.streaming.backpressure.initialRate</code></td>
  <td>not set</td>
  <td>
    This is the initial maximum receiving rate at which each receiver will receive data for the
    first batch when the backpressure mechanism is enabled.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.blockInterval</code></td>
  <td>200ms</td>
  <td>
    Interval at which data received by Spark Streaming receivers is chunked
    into blocks of data before storing them in Spark. Minimum recommended - 50 ms. See the
    <a href="streaming-programming-guide.html#level-of-parallelism-in-data-receiving">performance
     tuning</a> section in the Spark Streaming programing guide for more details.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.maxRate</code></td>
  <td>not set</td>
  <td>
    Maximum rate (number of records per second) at which each receiver will receive data.
    Effectively, each stream will consume at most this number of records per second.
    Setting this configuration to 0 or a negative number will put no limit on the rate.
    See the <a href="streaming-programming-guide.html#deploying-applications">deployment guide</a>
    in the Spark Streaming programing guide for mode details.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.writeAheadLog.enable</code></td>
  <td>false</td>
  <td>
    Enable write ahead logs for receivers. All the input data received through receivers
    will be saved to write ahead logs that will allow it to be recovered after driver failures.
    See the <a href="streaming-programming-guide.html#deploying-applications">deployment guide</a>
    in the Spark Streaming programing guide for more details.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.unpersist</code></td>
  <td>true</td>
  <td>
    Force RDDs generated and persisted by Spark Streaming to be automatically unpersisted from
    Spark's memory. The raw input data received by Spark Streaming is also automatically cleared.
    Setting this to false will allow the raw data and persisted RDDs to be accessible outside the
    streaming application as they will not be cleared automatically. But it comes at the cost of
    higher memory usage in Spark.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.stopGracefullyOnShutdown</code></td>
  <td>false</td>
  <td>
    If <code>true</code>, Spark shuts down the <code>StreamingContext</code> gracefully on JVM
    shutdown rather than immediately.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.kafka.maxRatePerPartition</code></td>
  <td>not set</td>
  <td>
    Maximum rate (number of records per second) at which data will be read from each Kafka
    partition when using the new Kafka direct stream API. See the
    <a href="streaming-kafka-integration.html">Kafka Integration guide</a>
    for more details.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.kafka.maxRetries</code></td>
  <td>1</td>
  <td>
    Maximum number of consecutive retries the driver will make in order to find
    the latest offsets on the leader of each partition (a default value of 1
    means that the driver will make a maximum of 2 attempts). Only applies to
    the new Kafka direct stream API.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.ui.retainedBatches</code></td>
  <td>1000</td>
  <td>
    How many batches the Spark Streaming UI and status APIs remember before garbage collecting.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.driver.writeAheadLog.closeFileAfterWrite</code></td>
  <td>false</td>
  <td>
    Whether to close the file after writing a write ahead log record on the driver. Set this to 'true'
    when you want to use S3 (or any file system that does not support flushing) for the metadata WAL
    on the driver.
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.writeAheadLog.closeFileAfterWrite</code></td>
  <td>false</td>
  <td>
    Whether to close the file after writing a write ahead log record on the receivers. Set this to 'true'
    when you want to use S3 (or any file system that does not support flushing) for the data WAL
    on the receivers.
  </td>
</tr>
</table>

#### SparkR
<table class="table">
<tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
<tr>
  <td><code>spark.r.numRBackendThreads</code></td>
  <td>2</td>
  <td>
    Number of threads used by RBackend to handle RPC calls from SparkR package.
  </td>
</tr>
<tr>
  <td><code>spark.r.command</code></td>
  <td>Rscript</td>
  <td>
    Executable for executing R scripts in cluster modes for both driver and workers.
  </td>
</tr>
<tr>
  <td><code>spark.r.driver.command</code></td>
  <td>spark.r.command</td>
  <td>
    Executable for executing R scripts in client modes for driver. Ignored in cluster modes.
  </td>
</tr>
</table>

#### Deploy

<table class="table">
  <tr><th>Property Name</th><th>Default</th><th>Meaning</th></tr>
  <tr>
    <td><code>spark.deploy.recoveryMode</code></td>
    <td>NONE</td>
    <td>The recovery mode setting to recover submitted Spark jobs with cluster mode when it failed and relaunches.
    This is only applicable for cluster mode when running with Standalone or Mesos.</td>
  </tr>
  <tr>
    <td><code>spark.deploy.zookeeper.url</code></td>
    <td>None</td>
    <td>When `spark.deploy.recoveryMode` is set to ZOOKEEPER, this configuration is used to set the zookeeper URL to connect to.</td>
  </tr>
  <tr>
    <td><code>spark.deploy.zookeeper.dir</code></td>
    <td>None</td>
    <td>When `spark.deploy.recoveryMode` is set to ZOOKEEPER, this configuration is used to set the zookeeper directory to store recovery state.</td>
  </tr>
</table>


#### Cluster Managers
Each cluster manager in Spark has additional configuration options. Configurations
can be found on the pages for each mode:

##### [YARN](running-on-yarn.html#configuration)

##### [Mesos](running-on-mesos.html#configuration)

##### [Standalone Mode](spark-standalone.html#cluster-launch-scripts)

# Environment Variables

Certain Spark settings can be configured through environment variables, which are read from the
`conf/spark-env.sh` script in the directory where Spark is installed (or `conf/spark-env.cmd` on
Windows). In Standalone and Mesos modes, this file can give machine specific information such as
hostnames. It is also sourced when running local Spark applications or submission scripts.

Note that `conf/spark-env.sh` does not exist by default when Spark is installed. However, you can
copy `conf/spark-env.sh.template` to create it. Make sure you make the copy executable.

The following variables can be set in `spark-env.sh`:


<table class="table">
  <tr><th style="width:21%">Environment Variable</th><th>Meaning</th></tr>
  <tr>
    <td><code>JAVA_HOME</code></td>
    <td>Location where Java is installed (if it's not on your default <code>PATH</code>).</td>
  </tr>
  <tr>
    <td><code>PYSPARK_PYTHON</code></td>
    <td>Python binary executable to use for PySpark in both driver and workers (default is <code>python2.7</code> if available, otherwise <code>python</code>).</td>
  </tr>
  <tr>
    <td><code>PYSPARK_DRIVER_PYTHON</code></td>
    <td>Python binary executable to use for PySpark in driver only (default is <code>PYSPARK_PYTHON</code>).</td>
  </tr>
  <tr>
    <td><code>SPARKR_DRIVER_R</code></td>
    <td>R binary executable to use for SparkR shell (default is <code>R</code>).</td>
  </tr>
  <tr>
    <td><code>SPARK_LOCAL_IP</code></td>
    <td>IP address of the machine to bind to.</td>
  </tr>
  <tr>
    <td><code>SPARK_PUBLIC_DNS</code></td>
    <td>Hostname your Spark program will advertise to other machines.</td>
  </tr>
</table>

In addition to the above, there are also options for setting up the Spark
[standalone cluster scripts](spark-standalone.html#cluster-launch-scripts), such as number of cores
to use on each machine and maximum memory.

Since `spark-env.sh` is a shell script, some of these can be set programmatically -- for example, you might
compute `SPARK_LOCAL_IP` by looking up the IP of a specific network interface.

Note: When running Spark on YARN in `cluster` mode, environment variables need to be set using the `spark.yarn.appMasterEnv.[EnvironmentVariableName]` property in your `conf/spark-defaults.conf` file.  Environment variables that are set in `spark-env.sh` will not be reflected in the YARN Application Master process in `cluster` mode.  See the [YARN-related Spark Properties](running-on-yarn.html#spark-properties) for more information.

# Configuring Logging

Spark uses [log4j](http://logging.apache.org/log4j/) for logging. You can configure it by adding a
`log4j.properties` file in the `conf` directory. One way to start is to copy the existing
`log4j.properties.template` located there.

# Overriding configuration directory

To specify a different configuration directory other than the default "SPARK_HOME/conf",
you can set SPARK_CONF_DIR. Spark will use the configuration files (spark-defaults.conf, spark-env.sh, log4j.properties, etc)
from this directory.

# Inheriting Hadoop Cluster Configuration

If you plan to read and write from HDFS using Spark, there are two Hadoop configuration files that
should be included on Spark's classpath:

* `hdfs-site.xml`, which provides default behaviors for the HDFS client.
* `core-site.xml`, which sets the default filesystem name.

The location of these configuration files varies across CDH and HDP versions, but
a common location is inside of `/etc/hadoop/conf`. Some tools, such as Cloudera Manager, create
configurations on-the-fly, but offer a mechanisms to download copies of them.

To make these files visible to Spark, set `HADOOP_CONF_DIR` in `$SPARK_HOME/spark-env.sh`
to a location containing the configuration files.