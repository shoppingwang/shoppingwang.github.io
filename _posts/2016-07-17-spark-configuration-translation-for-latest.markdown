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
    每个reduce任务同时获取map输出数据的最大大小。由于每一个输出需要我们创建一个缓冲区来接收数据，这就意味着每一个reduce任务需要一个额外固定的内存，所以保持它为比较小的值直到你拥有较多的使用内存的时候。
  </td>
</tr>
<tr>
  <td><code>spark.reducer.maxReqsInFlight</code></td>
  <td>Int.MaxValue</td>
  <td>
    这个配置限制了在某一给定的时间点，远程请求能获取的最大的block数量。当集群中的主机数量增加的时候，它将导致某一个或某些节点的接入请求非常大，然后工作节点因不堪重负会挂掉。通过限制fetch请求的连接数量，可以减轻这一状况的发生。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.compress</code></td>
  <td>true</td>
  <td>
    是否压缩map的输出文件。这通常是一个比较好的主意。压缩将会使用<code>spark.io.compression.codec</code>的指定值。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.file.buffer</code></td>
  <td>32k</td>
  <td>
    针对每一个shuffle文件的输出的内存缓冲区大小。这些缓冲区在创建中间shuffle文件时能够减少磁盘寻道和系统调用次数。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.maxRetries</code></td>
  <td>3</td>
  <td>
    (Netty only) 如果这个值设置为一个非0值，那么当由于IO错误引发的数据获取失败时，fetch操作将自动重试。在面对长的GC暂定或者瞬时的网络连接问题时，这个尝试逻辑有助于巨大的shuffle操作稳定。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.numConnectionsPerPeer</code></td>
  <td>1</td>
  <td>
    (Netty only) 在大的集群中主机之间的连接被复用可以减少建立连接的消耗。针对主机中有许多的硬盘和较少的主机，这将导致所有硬盘不饱和的并发，所以用户应该考虑增加这个值。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.preferDirectBufs</code></td>
  <td>true</td>
  <td>
    (Netty only) 被用来shuffle和缓存block传输中的堆外缓冲，以止减少垃圾收集器的工作。在许多环境中堆外内存都是被严格限制的，用户可能希望关闭这一特性来强制使Netty分配的内存都在堆上。  </td>
</tr>
<tr>
  <td><code>spark.shuffle.io.retryWait</code></td>
  <td>5s</td>
  <td>
    (Netty only) 在进行fetch操作重试时等待多长时间。最大的延迟默认是重试15秒，计算公式为<code>maxRetries * retryWait</code>。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.service.enabled</code></td>
  <td>false</td>
  <td>
    是否开启外部的shuffle服务。这个服务保存由executor写入的shuffle文件，所以executor能被安全的移除。这个特性必须被打开当<code>spark.dynamicAllocation.enabled</code>设置为"true"时。为了启用它，外部shuffle服务必须设置。参见<a href="job-scheduling.html#configuration-and-setup">动态资源分配和设置</a>获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.service.port</code></td>
  <td>7337</td>
  <td>
    外部shuffle服务运行的端口号。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.sort.bypassMergeThreshold</code></td>
  <td>200</td>
  <td>
    (Advanced) 在依赖排序的shuffle管理器中，如果没有map端的聚合并且有很多的reduce分区，避免合并-排序数据。
  </td>
</tr>
<tr>
  <td><code>spark.shuffle.spill.compress</code></td>
  <td>true</td>
  <td>
    当在shuffle操作溢出到磁盘时是否开启压缩。压缩将会使用<code>spark.io.compression.codec</code>属性的值。
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
    若<code>spark.eventLog.enabled</code>为true，则表示是否压缩事件日志。
  </td>
</tr>
<tr>
  <td><code>spark.eventLog.dir</code></td>
  <td>file:///tmp/spark-events</td>
  <td>
    若<code>spark.eventLog.enabled</code>为true，则表示Spark事件日志的基目录。在这个基目录下，Spark为每一个应用创建一个日志子目录，并且在这个目录中记录特定的应用日志事件。用户可能想要设置一个统一的目录位置，例如HDFS目录，这样就能通过历史服务器读取历史日志了。
  </td>
</tr>
<tr>
  <td><code>spark.eventLog.enabled</code></td>
  <td>false</td>
  <td>
    决定是否开启Spark事件日志记录，应用完成后，在WEB UI有助于重现应用状况。
  </td>
</tr>
<tr>
  <td><code>spark.ui.killEnabled</code></td>
  <td>true</td>
  <td>
    是否允许从WEB UI终止掉stages和相关的jobs。
  </td>
</tr>
<tr>
  <td><code>spark.ui.port</code></td>
  <td>4040</td>
  <td>
    应用程序画板的端口号，用来展示使用的内存和工作负载数据。
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedJobs</code></td>
  <td>1000</td>
  <td>
    在垃圾收集之前，允许多少个job的Spark UI和status API保留。
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedStages</code></td>
  <td>1000</td>
  <td>
    在垃圾收集之前，保留多少个stage的Spark UI和status API.
    collecting.
  </td>
</tr>
<tr>
  <td><code>spark.worker.ui.retainedExecutors</code></td>
  <td>1000</td>
  <td>
    在垃圾收集之前，保留多少个已经完成的executor的Spark UI和status API。
  </td>
</tr>
<tr>
  <td><code>spark.worker.ui.retainedDrivers</code></td>
  <td>1000</td>
  <td>
    在垃圾收集之前，保留多少个已经完成的driver的Spark UI和status API。
  </td>
</tr>
<tr>
  <td><code>spark.sql.ui.retainedExecutions</code></td>
  <td>1000</td>
  <td>
    在垃圾收集之前，保留多少个执行完成的execution的Spark UI和status API。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.ui.retainedBatches</code></td>
  <td>1000</td>
  <td>
    在垃圾收集之前，保留多少个batch的Spark UI和status API。
  </td>
</tr>
<tr>
  <td><code>spark.ui.retainedDeadExecutors</code></td>
  <td>100</td>
  <td>
    在垃圾收集之前，保留多少个已经挂掉的executor的Spark UI和status API。
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
    在发送广播变量前是否进行压缩，通常这是一个不错的主意。
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.codec</code></td>
  <td>lz4</td>
  <td>
    内部数据使用的压缩方式，比如RDD的分区、广播变量和shuffle的输出。默认，Spark提供三种压缩方式：<code>lz4</code>, <code>lzf</code>和<code>snappy</code>。你也可能使用类全名指定特定的压缩器，例如：
    <code>org.apache.spark.io.LZ4CompressionCodec</code>,
    <code>org.apache.spark.io.LZFCompressionCodec</code>,
    和<code>org.apache.spark.io.SnappyCompressionCodec</code>。
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.lz4.blockSize</code></td>
  <td>32k</td>
  <td>
    当LZ4压缩启用时，在LZ4的压缩方式中块的大小。当LZ4压缩开启时，减少这个块的大小也将减小shuffle的内存使用。
  </td>
</tr>
<tr>
  <td><code>spark.io.compression.snappy.blockSize</code></td>
  <td>32k</td>
  <td>
    当Snappy压缩启用时，在Snappy的压缩方式中块的大小。当Snappy压缩开启时，减少这个块的大小也将减小shuffle的内存使用。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.classesToRegister</code></td>
  <td>(none)</td>
  <td>
    如果你使用Kyro序列化，使用逗号分隔需要注册到Kyro的自定义类列表。参见<a href="tuning.html#data-serialization">调调优</a>获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.referenceTracking</code></td>
  <td>true (false when using Spark SQL Thrift Server)</td>
  <td>
    当开启使用Kyro序列化时，是否跟踪同一个对像的引用，在你的对象图中有循环或者它们包含了多个对象的副本时是非常有用的。若你知道不会有这个的情况，那么关闭这个特性将会提升你应用程序的性能。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.registrationRequired</code></td>
  <td>false</td>
  <td>
    使用Kryo序列化时是否需要进行注册。如果设置为true，若未注册的类需要序列化，Kyro将会抛出异常。如果设置为false（默认值），Kryo将会对未注册类的每个对象写入对应的类全名。写入类名称将会有显著的性能开销，所以开启这个选项将严格的强制用户不能忽略类的注册。
  </td>
</tr>
<tr>
  <td><code>spark.kryo.registrator</code></td>
  <td>(none)</td>
  <td>
    如果你使用Kryo序列化，使用逗号分隔的自定义类列表来向Kryo注册你的自定义类。当你需要使用你自己的方式注册自己的自定义类时，这个属性就会很有用，例如：为了指定一个自定义字段序列化器。其他情况下，使用<code>spark.kryo.classesToRegister</code>是更简单的。若要设置这个属性，那么类应该继承自
    <a href="api/scala/index.html#org.apache.spark.serializer.KryoRegistrator">
    <code>KryoRegistrator</code></a>.
    参见<a href="tuning.html#data-serialization">调优指南</a>获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.kryoserializer.buffer.max</code></td>
  <td>64m</td>
  <td>
    Kryo序列化缓冲区允许的最大大小。这个缓冲区必须比尝试序列化的最大对象还要大。当你在Kryo中获取到一个"buffer limit exceeded"的异常，你就应该增加这个值。
  </td>
</tr>
<tr>
  <td><code>spark.kryoserializer.buffer</code></td>
  <td>64k</td>
  <td>
    Kryo序列化缓冲区初始化大小。注意这是每一个核一个缓冲区。如果需要的话，这个缓冲区将会增长到
     <code>spark.kryoserializer.buffer.max</code>属性配置的大小。
  </td>
</tr>
<tr>
  <td><code>spark.rdd.compress</code></td>
  <td>false</td>
  <td>
    是否序列化RDD的分区。(例如：属性
    <code>StorageLevel.MEMORY_ONLY_SER</code>在Java和Scala或者 <code>StorageLevel.MEMORY_ONLY</code>在PYTHON中).
    在浪费一些额外的CPU周期的前提下可以实质性的减少内存空间的使用。
  </td>
</tr>
<tr>
  <td><code>spark.serializer</code></td>
  <td>
    org.apache.spark.serializer.KryoSerializer
  </td>
  <td>
  指定用来序列化的类库，包括通过网络传输数据或缓存数据时的序列化。默认的Java序列化对于任何可以被序列化的Java对象都适用，但是速度很慢。我们推荐在追求速度时使用org.apache.spark.serializer.KryoSerializer并对Kryo进行适当的调优。该项可以配置为任何org.apache.spark.Serializer的子类。
  </td>
</tr>
<tr>
  <td><code>spark.serializer.objectStreamReset</code></td>
  <td>100</td>
  <td>
    当使用org.apache.spark.serializer.JavaSerializer进行序列化时，序列化器缓存这些对象来防止写入重复数据，然而也阻止了垃圾收集器收集到这些对象。通过调用‘reset’，你可以在序列化器中刷出那些对象，并且允许旧的对象被回收。关闭这个重置调用间隔可以把这个属性值设置为-1。默认的，每100个对象序列化器将会被重置。
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
    用来执行和存储占（heap space - 300MB）内存的比例。这个比例越低，那么将会发生更频繁的磁盘溢出和更多的磁盘数据交换。这个配置项的目的是在于为内部元数据、用户数据结构和在很少的、不经常使用的大的无法精确估计记录的大小。推荐保持这个配置项的默认值。想要获取更多细节，包括当增加这个值时关于如何正确的对JVM的垃圾收集进行调优参见
    <a href="tuning.html#memory-management-overview">描述</a>。
  </td>
</tr>
<tr>
  <td><code>spark.memory.storageFraction</code></td>
  <td>0.5</td>
  <td>
    为防止内存数据被清除而设置的总的能被使用的存储内存大小，它的比例大小设置不依赖于<code>s​park.memory.fraction</code>。这个值设置得越大，那执行任务可用的内存将会更小，也将会导致数据更频繁的向磁盘溢出。推荐保持这个配置项的默认值。想要获取更多细节，参见
    <a href="tuning.html#memory-management-overview">描述</a>。
  </td>
</tr>
<tr>
  <td><code>spark.memory.offHeap.enabled</code></td>
  <td>false</td>
  <td>
    如果为true，Spark将会尝试使用堆外内存来做确定的操作。如果堆外内存使用是被开启的，那么这个参数<code>spark.memory.offHeap.size</code>的值必须大于0。
  </td>
</tr>
<tr>
  <td><code>spark.memory.offHeap.size</code></td>
  <td>0</td>
  <td>
    堆外内存分配的绝对大小，以bytes为单位。这个参数已经对堆内存的利用没有影响了，所以如果你的executor的总内存消耗是被严格限制的，确保能正确的缩小你的JVM堆内存的大小。 当 <code>spark.memory.offHeap.enabled=true</code>时，这个值必须大于0。
  </td>
</tr>
<tr>
  <td><code>spark.memory.useLegacyMode</code></td>
  <td>false</td>
  <td>
    是否在Spark 1.5及之前的版本中使用旧的内存管理机制。旧的机制会严格的对堆内存大小进行固定的大小分区，如果应用没有进行调优，这将会导致潜在的过多的数据溢出。在这个选项开启前，下面的这些配置是未生效的：
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
    <code>TorrentBroadcastFactory</code>的每一个分块的大小。
    如果太大，在进行广播时将会降低并发度；然而，如果太小，<code>BlockManager</code>也可能会带来性能的影响。
  </td>
</tr>
<tr>
  <td><code>spark.executor.cores</code></td>
  <td>
    YARN模式中为1个，在standalone和Mesos的粗粒度模式中是所有的可用的核数。
  </td>
  <td>
    每个executor可用的核的数量。

    在standalone和Mesos粗粒度模式中，设置这个值可以让多个executor同时跑在一个work上，依赖于运行的worker有足够的核数来运行多个executor。否则，每个应用只有一个executor运行在每个worker上。
  </td>
</tr>
<tr>
  <td><code>spark.default.parallelism</code></td>
  <td>
    针对分布式的shuffle操作，比如<code>reduceByKey</code>和<code>join</code>，在它们的父RDD中有着大量的分区。针对像<code>parallelize</code>没有父RDD的操作，它的行为将依赖于集群管理器：
    <ul>
      <li>本地模式：本地机器的核心数量</li>
      <li>Mesos的细粒度模式：8</li>
      <li>其他：所有executor节点可能核心的数量或者为2，哪个值大用哪个</li>
    </ul>
  </td>
  <td>
    在进行<code>join</code>,
    <code>reduceByKey</code>和<code>parallelize</code>的转换操作时，若用户没有设置并行度，将使用此处理设置的默认并行度。
  </td>
</tr>
<tr>
    <td><code>spark.executor.heartbeatInterval</code></td>
    <td>10s</td>
    <td>每个executor汇报给驱动的心跳间隔时间。心跳可以让驱动知道哪些executor依然活着并且在处理进度属性中更新它的任务状态。</td>
</tr>
<tr>
  <td><code>spark.files.fetchTimeout</code></td>
  <td>60s</td>
  <td>
    当从driver调用SparkContext.addFile()增加文件时的超时时间。
  </td>
</tr>
<tr>
  <td><code>spark.files.useFetchCache</code></td>
  <td>true</td>
  <td>
    如果设置为true（默认），文件的获取将会使用属于同一个应用的被许多executor共享的本地缓存。当许多executor运行在同一台主机时，这样能提升任务运行的效率。如果设置为false，这些缓存优化将会不起作用，并且所有的executor将会获取他们自己的文件的拷贝。为了使用Spark在NFS文件系统上的本地目录，这种优化可能会被禁用。(参见
    <a href="https://issues.apache.org/jira/browse/SPARK-6313">SPARK-6313</a>获取更多信息)。
  </td>
</tr>
<tr>
  <td><code>spark.files.overwrite</code></td>
  <td>false</td>
  <td>
    当文件已经存在时并且它的内存和源文件的内容不匹配，是否覆写已经通过SparkContext.addFile()增加的文件。
  </td>
</tr>
<tr>
    <td><code>spark.hadoop.cloneConf</code></td>
    <td>false</td>
    <td>如果设置为true，会为每一个任务克隆一份Hadoop<code>Configuration</code>。这个选项应该在需要使用<code>Configuration</code>有线程安全问题时打开(参见<a href="https://issues.apache.org/jira/browse/SPARK-2546">SPARK-2546</a>获取更多细节)。这个选项默认是关闭的，是为了避免任务的不可预期的性能下降而又不是由这个问题引起的。
    </td>
</tr>
<tr>
    <td><code>spark.hadoop.validateOutputSpecs</code></td>
    <td>true</td>
    <td>如果设置为true，将会在saveAsHadoopFile和其他变化中的检查输出的合法性(例如，检查输出目录是否已经存在)。如果想让程序在遇到已经存在的目录只打印异常信息时，可以将此选项禁用掉。我们建议用户不要禁用此选项，除非是为了尝试和之前的Spark版本进行适配。最简单的方式就是顺手使用Hadoop的文件系统API删除已经存在的目录。这个选项在Spark Streaming的StreamingContext生成任务的情下会被忽略，由于检查点恢复机制，因为数据可能已经被写入已经存在的输出目录了。
    </td>
</tr>
<tr>
  <td><code>spark.storage.memoryMapThreshold</code></td>
  <td>2m</td>
  <td>
    当从磁盘读取至少多大的数据再进行map内存映射操作。这将阻止Spark在非常小的块上进行内存中的mapping操作。通常情况下，内存映射会因为块数据的关闭或者把数据扇进系统页会有较高的系统开销。
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
    被允许在“控制面板”沟通过程中使用的最大的消息大小（MB）；通常只是应用在executor和driver之间发送map输出大小信息上。如果你正在运行的任务有几千个map和reduce任务增加这个属性的值，并且查看RPC消息的大小。
  </td>
</tr>
<tr>
  <td><code>spark.blockManager.port</code></td>
  <td>(random)</td>
  <td>
    所有的块管理器的监听端口。这些既存在于driver端，也存在于executor端。
  </td>
</tr>
<tr>
  <td><code>spark.driver.host</code></td>
  <td>(local hostname)</td>
  <td>
    Driver需要监听的主机名或者IP地址。
    这个通常用于与executor和standalone Master进行沟通。
    </td>
</tr>
<tr>
  <td><code>spark.driver.port</code></td>
  <td>(random)</td>
  <td>
    Driver的监听端口。
    这个通常用于与executor和standalone Master进行沟通。
  </td>
</tr>
<tr>
  <td><code>spark.network.timeout</code></td>
  <td>120s</td>
  <td>
    默认的所有的网络交互的超时时间。若如下参数未进行配置，这个配置将会被以下配置中所使用：
    <code>spark.core.connection.ack.wait.timeout</code>,
    <code>spark.storage.blockManagerSlaveTimeoutMs</code>,
    <code>spark.shuffle.io.connectionTimeout</code>, <code>spark.rpc.askTimeout</code> or
    <code>spark.rpc.lookupTimeout</code>。
  </td>
</tr>
<tr>
  <td><code>spark.port.maxRetries</code></td>
  <td>16</td>
  <td>
    尝试绑定一个端口的最大重试次数。当一个端口被给定的值为非0值时，每一次后续的重试之前都会在之前重试的端口的值上加1。这个本质上是让它能够尝试从一个起始端口到起始端口+重试次数的端口范围内进行重试。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.numRetries</code></td>
  <td>3</td>
  <td>
    RPC任务在失败放弃之前的最大重试次数。
    一个RPC任务最多会运行此选项指定的次数。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.retry.wait</code></td>
  <td>3s</td>
  <td>
    RPC等待最长的响应时间，否则进行重试。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.askTimeout</code></td>
  <td>120s</td>
  <td>
    RPC请求响应的最大超时时间。
  </td>
</tr>
<tr>
  <td><code>spark.rpc.lookupTimeout</code></td>
  <td>120s</td>
  <td>
    RPC远程端点查找操作的最大等待超时时间。
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
    当程序运行在一个<a href="spark-standalone.html">standalone集群部署模式</a>或者一个
    <a href="running-on-mesos.html#mesos-run-modes">Mesos集群粗粒度模式</a>时，应用程序可以从集群申请到的最大的CPU核心数量（不是每台机器）。如果没有设置，默认值在Spark的standalone集群模式下将会是
    <code>spark.deploy.defaultCores</code>指定的值，或者在Mesos下为无限（所有可用的核心）。 
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait</code></td>
  <td>3s</td>
  <td>
    在缺少本地化节点时启动一个数据本地化的任务前等待多久后放弃。同样的等待设置将会逐个应用于多个本地化级别（进程本地化、节点本地化、机架本地化和其他）。你也可以对每一种级别单独自定义设置等待时间，例如<code>spark.locality.wait.node</code>等等。如果你的任务很长并且很少本地化，那么你可以增加此设置的时间，但是通常情况下这也工作得很好。
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.node</code></td>
  <td>spark.locality.wait</td>
  <td>
    自定义任务节点本地化的等待时间。例如，你可以设置这项为0来跳过节点的本地化并且立即搜索机架 的本地化（假设你的集群有机架的信息）。
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.process</code></td>
  <td>spark.locality.wait</td>
  <td>
    自定义进程本地化等待时间。这个参数尝试在一个特定的executor进程中获取缓存数据。
  </td>
</tr>
<tr>
  <td><code>spark.locality.wait.rack</code></td>
  <td>spark.locality.wait</td>
  <td>
    自定义机架本地化等待时间。
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.maxRegisteredResourcesWaitingTime</code></td>
  <td>30s</td>
  <td>
    在调度开始之前等待资源注册的最大时间。
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.minRegisteredResourcesRatio</code></td>
  <td>YARN模式为0.8;standalone械和Mesos粗粒度模式为0.0</td>
  <td>
    在任务调度开始前等待最小的注册资源比例(注册的资源/期望得到的资源)
    (在YARN模式下的资源是指executor，在standalone模式和Mesos粗粒度模式下是CPU的核心数['spark.cores.max'的值是在Mesos粗粒度模式下总的期望得到的资源] )无论最小的资源比例是否已经获得，在调度开始前仍然会等待由<code>spark.scheduler.maxRegisteredResourcesWaitingTime</code>指定的时间。
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.mode</code></td>
  <td>FIFO</td>
  <td>
    提交给同一个SparkContext任务的<a href="job-scheduling.html#scheduling-within-an-application">调度模式</a>。此属性能够被设置为<code>FAIR</code>模式来资源进行公平调度替代队列的方式。在多用户服务下有用。
  </td>
</tr>
<tr>
  <td><code>spark.scheduler.revive.interval</code></td>
  <td>1s</td>
  <td>
    调度器接收worker提供运行任务资源的时间间隔。
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
    检查任务推测执行的时间间隔。
  </td>
</tr>
<tr>
  <td><code>spark.speculation.multiplier</code></td>
  <td>1.5</td>
  <td>
    当任务慢于多少个任务之后考虑使用推测执行。
  </td>
</tr>
<tr>
  <td><code>spark.speculation.quantile</code></td>
  <td>0.75</td>
  <td>
    对于一个特定的stage在开始执行推测任务之前，任务必须完成的百分比数。
  </td>
</tr>
<tr>
  <td><code>spark.task.cpus</code></td>
  <td>1</td>
  <td>
    每个任务分配的CPU核心数。
  </td>
</tr>
<tr>
  <td><code>spark.task.maxFailures</code></td>
  <td>4</td>
  <td>
    JOB的每一个单独的任务在放弃前允许失败的次数。这个值应该大于或者等于1.允许重试的次数=这个值 - 1。
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
    是否启用动态资源分配，使应用基于其负载调整注册的executor个数。更多信息，参见描述
    <a href="job-scheduling.html#dynamic-resource-allocation">这里</a>。
    <br><br>
    使用这个选项需要设置<code>spark.shuffle.service.enabled</code>这个参数。
    下面是一些相关的配置选项:
    <code>spark.dynamicAllocation.minExecutors</code>,
    <code>spark.dynamicAllocation.maxExecutors</code>, 和
    <code>spark.dynamicAllocation.initialExecutors</code>
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.executorIdleTimeout</code></td>
  <td>60s</td>
  <td>
    如果开启动态资源分配，并且executor已经超出配置的闲置时间，那么此executor将会被移除。更多信息，参见
    <a href="job-scheduling.html#resource-allocation-policy">描述</a>。
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.cachedExecutorIdleTimeout</code></td>
  <td>infinity</td>
  <td>
    如果开启动态资源分配，如果一个缓存了数据块的executor闲置的时间超过了此配置的时间，那么此executor将会被移除。更多信息，参见
    <a href="job-scheduling.html#resource-allocation-policy">描述</a>。
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.initialExecutors</code></td>
  <td><code>spark.dynamicAllocation.minExecutors</code></td>
  <td>
    在开启动态资源分配的情况下，初始化指定数量的eecutor。
    <br /><br />
    如果设置了`--num-executors` (or `spark.executor.instances`)参数，如果设置的值大于这个值，那么它将使用此配置项的值来初始化executor个数。
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.maxExecutors</code></td>
  <td>infinity</td>
  <td>
    在开启动态资源分配开启的情况下，应用能申请到的executor数量的上限。
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.minExecutors</code></td>
  <td>0</td>
  <td>
    在开启动态资源分配开启的情况下，应用能申请到的executor数量的下限。
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.schedulerBacklogTimeout</code></td>
  <td>1s</td>
  <td>
    在开启动态资源分配开启的情况下，如果有等待任务的时间超过此时间值，则会申请新的executor执行任务。更多信息，参见<a href="job-scheduling.html#resource-allocation-policy">描述</a>。
  </td>
</tr>
<tr>
  <td><code>spark.dynamicAllocation.sustainedSchedulerBacklogTimeout</code></td>
  <td><code>schedulerBacklogTimeout</code></td>
  <td>
    和<code>spark.dynamicAllocation.schedulerBacklogTimeout</code>参数的意义一样，但是只是用于子executor的请求。更多信息，参见
    <a href="job-scheduling.html#resource-allocation-policy">描述</a>。
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
    是否开启Spark acls。如果开启，将会检查用户是否有察看或者修改JOB的权限。注意这个要求用户必须是已知的，如果用户的身份为空，那么将不会对用户身份做检查。可以在UI上进行过滤来设置用户的身份来解决这个问题。
  </td>
</tr>
<tr>
  <td><code>spark.admin.acls</code></td>
  <td>Empty</td>
  <td>
    以逗号进行分隔的可以查看全部任务和修改所有任务的用户/管理员列表。这个可以用在当你的任务运行在一个共享的集群上，当你的任务不生效时，会有很多的管理员和开发者来帮你调试任务。如果设置为*则表示任务一个用户都具有管理员的权限。
  </td>
</tr>
<tr>
  <td><code>spark.admin.acls.groups</code></td>
  <td>Empty</td>
  <td>
    以逗号进行分隔的可以查看全部任务和修改所有任务的组列表。这在当你有一系列的管理者和开发者帮你维护和调试底层基础架构时有用。如果设置为*则表示任务一个用户都具有管理员的权限。这个用户组的列表是从<code>spark.user.groups.mapping</code>属性中指定的属性映射中获取的。检查这个配置<code>spark.user.groups.mapping</code>获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.user.groups.mapping</code></td>
  <td><code>org.apache.spark.security.ShellBasedGroupsMappingProvider</code></td>
  <td>
    一个实现了org.apache.spark.security.GroupMappingServiceProvider接口的组的映射配置，决定用户所属组的列表。默认提供的是一个unix shell的实现<code>org.apache.spark.security.ShellBasedGroupsMappingProvider</code>，这个可能为某一个用户获取他所属的组列表。
    <em>注意：</em>这个实现只支持在UNIX/LINUX下进行配置。WINDOWS环境下暂时还不支持。然而，你可以实现<code>org.apache.spark.security.GroupMappingServiceProvider</code>接口来支持一个新的平台/协议。
  </td>
</tr>
<tr>
  <td><code>spark.authenticate</code></td>
  <td>false</td>
  <td>
    是否加密spark的内部连接。如果没运行在YARN上，参见
    <code>spark.authenticate.secret</code>。
  </td>
</tr>
<tr>
  <td><code>spark.authenticate.secret</code></td>
  <td>None</td>
  <td>
    设置SPARK在不同的容器之间的安全KEY。如果没有运行在YARN上并且鉴权是启用的，那么必须设置这个属性。
  </td>
</tr>
<tr>
  <td><code>spark.authenticate.enableSaslEncryption</code></td>
  <td>false</td>
  <td>
    当鉴权开启后开启加密通讯。这个选项现在只支持block传输服务。
  </td>
</tr>
<tr>
  <td><code>spark.network.sasl.serverAlwaysEncrypt</code></td>
  <td>false</td>
  <td>
    在支持SASL鉴权情况下，禁止服务间的未加密的连接。当前外部shuffle服务支持此特性。
  </td>
</tr>
<tr>
  <td><code>spark.core.connection.ack.wait.timeout</code></td>
  <td>60s</td>
  <td>
    在连接等待响应发生前的超时和放弃的时间。为了避免意外的由于较长时间GC带来的超时，你可以设置一个较大的值。
  </td>
</tr>
<tr>
  <td><code>spark.core.connection.auth.wait.timeout</code></td>
  <td>30s</td>
  <td>
    在连接等待鉴权发生前的超时和放弃的时间。
  </td>
</tr>
<tr>
  <td><code>spark.modify.acls</code></td>
  <td>Empty</td>
  <td>
    具有访问Spark Job权限的用户列表，以逗号分隔。默认的只有启动Spark Job的用户才会有修改（例如kill）掉自己Job的权限。如果设置为*则表示任务用户都有权限对Job进行修改。
  </td>
</tr>
<tr>
  <td><code>spark.modify.acls.groups</code></td>
  <td>Empty</td>
  <td>
    具有访问Spark Job权限的组列表，以逗号分隔。这个可用于当你有一系列的管理员或者开发者来自同一个团队并且对Job具有相同的控制权限时。当设置为*时表示任何用户任何组都有修改Spark Job的权限。用户组映射的信息是从<code>spark.user.groups.mapping</code>属性中获取的。检查这个配置<code>spark.user.groups.mapping</code>以获取更多的信息。
  </td>
</tr>
<tr>
  <td><code>spark.ui.filters</code></td>
  <td>None</td>
  <td>
    以逗号分隔的应用在 Spark UI上的过滤器类列表。这个过滤器必须是标准的<a href="http://docs.oracle.com/javaee/6/api/javax/servlet/Filter.html">
    javax servlet过滤器</a>。每一个过滤器的参数可以指定可以指定一个Java系统参数：<br />
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
    以逗号分隔的有查看Spark web uiL界面的用户列表。默认情况下只有启动这个Spark job的用户才有相应的控制权限。设置为*时意味着任何用户都有权限访问这个Spark job。
  </td>
</tr>
<tr>
  <td><code>spark.ui.view.acls.groups</code></td>
  <td>Empty</td>
  <td>
    以逗号分隔的访问Spark web ui的组列表。这个可以用于当你有一个管理员或者开发者用户集合时能帮你监控Spark任务的提交。设置为*意味着任何组的任何用户可以查看Spark job的详情。用户的组映射信息是从<code>spark.user.groups.mapping</code>配置实体中获取的。检查这个配置
    <code>spark.user.groups.mapping</code>以获取更多的信息。
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
            <p>在所有支持SSL协议的情况下是否开启SSL连接。</p>

            <p>所有的SSL设置比如<code>spark.ssl.xxx</code>，这里的<code>xxx</code>是配置属性的一部分，注意支持所有协议的全局配置。为了覆盖某一个特定的协议，被覆盖的属性必须覆盖协议命名空间。</p>

            <p>使用<code>spark.ssl.YYY.XXX</code>设置来针对某一个特定的协议来覆盖全局配置项中的<code>YYY</code>。当前<code>YYY</code>只能应用于文件系统中的<code>fs</code>。</p>
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.enabledAlgorithms</code></td>
        <td>Empty</td>
        <td>
            以逗号分隔的密码列表。指定的密码必须是被JVM所支持的。协议引用列表可在
            <a href="https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https">这里</a>找到。
            注意：如果没有设置，将会使用JVM默认的密码套件。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyPassword</code></td>
        <td>None</td>
        <td>
            key-store中的私钥。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStore</code></td>
        <td>None</td>
        <td>
            key-store的文件路径。这个路径可以是绝对的或者是相对于组件的起始路径。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStorePassword</code></td>
        <td>None</td>
        <td>
            key-store的密码。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.keyStoreType</code></td>
        <td>JKS</td>
        <td>
            key-store的类型。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.protocol</code></td>
        <td>None</td>
        <td>
            协议的名称。此协议必须被JVM所支持。支持协议的引用列表可以在<a href="https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https">这里</a>
            找到。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.needClientAuth</code></td>
        <td>false</td>
        <td>
            如果SSL需要客户端鉴权则设置为true。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStore</code></td>
        <td>None</td>
        <td>
            trust-store文件的路径。这个路径可以是绝对路径或者相对于组件起始的相对路径。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStorePassword</code></td>
        <td>None</td>
        <td>
            trust-store的密码。
        </td>
    </tr>
    <tr>
        <td><code>spark.ssl.trustStoreType</code></td>
        <td>JKS</td>
        <td>
            trust-store的类型。
        </td>
    </tr>
</table>


#### Spark SQL
运行<code>SET -v</code>命令将会显示SQL的所有配置实体信息。

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
    开启或禁用Spark Streaming的内部背压机制（since 1.5）。这个可以让Spark Streaming控制接收数据的速率，这个速率是基于当前批处理调度的延迟和处理时间来的，所以系统会尽可能接收系统能够处理的数据。在内部，这个接收器动态的设置最大接收数据的速率。这个速率的上限是由
    <code>spark.streaming.receiver.maxRate</code>和<code>spark.streaming.kafka.maxRatePerPartition</code>
指定的(如下所示)。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.backpressure.initialRate</code></td>
  <td>not set</td>
  <td>
    当背压机制开启时，接收器第一次接收批处理时初始化最大的接收速率。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.blockInterval</code></td>
  <td>200ms</td>
  <td>
    Spark Streaming接收数据时间隔多长时间将收到的数据块存入Spark的block中。最小值建议是-50ms。参见Spark Streaming
    <a href="streaming-programming-guide.html#level-of-parallelism-in-data-receiving">性能调优</a>章节获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.maxRate</code></td>
  <td>not set</td>
  <td>
    每一个接收器接收数据最大的速率(每秒接收数据的数量)。
    实际上，每个流每秒将会消耗最多它能消耗的数据。将这个配置设置为0或者一个负数，那么数据接收速率将没有限制。
    参见Spark Streaming编程指引中的<a href="streaming-programming-guide.html#deploying-applications">部署指引</a>获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.writeAheadLog.enable</code></td>
  <td>false</td>
  <td>
    针对接收器开启WAL。所有通过接收器的数据将会先写入到日志中，这样允许当驱动错误发生时可以从日志中恢复数据。
    参见Spark Streaming编程指引中的<a href="streaming-programming-guide.html#deploying-applications">部署指引</a>获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.unpersist</code></td>
  <td>true</td>
  <td>
    强制Spark Streaming生成的RDD持久化并且能自动的从Spark的内存中自动去持久化。Spark Streaming原始接收的输入数据也将会被自动清除。将这个属性设置为false将会允许原始数据持久化并能被外部的streaming应用所访问，它也不会自动被清除。但是这将会在Spark中使用过高的内存。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.stopGracefullyOnShutdown</code></td>
  <td>false</td>
  <td>
    如果设置为<code>true</code>，比起直接停止的方式，Spark在JVM中将会优雅的停止。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.kafka.maxRatePerPartition</code></td>
  <td>not set</td>
  <td>
    当使用新的Kafka流API时，每一个Kafka分区中接收数据的最大速率（每秒接收数据的数量）。参见
    <a href="streaming-kafka-integration.html">Kafka集成指引</a>获取更多信息。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.kafka.maxRetries</code></td>
  <td>1</td>
  <td>
    针对Kafka中每个分区查找它的最新的offset信息的连续重试的最大次数（默认值为1意味着driver最多会做两次尝试）。此项只能应用于新的Kafka流API。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.ui.retainedBatches</code></td>
  <td>1000</td>
  <td>
    在垃圾收集之前，Spark Web UI中保存多少Spark Streaming UI和status APIs信息。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.driver.writeAheadLog.closeFileAfterWrite</code></td>
  <td>false</td>
  <td>
    在Driver写入一条WAL记录后是否关闭文件。当你想要使用S3（或者其他不支持flushing的文件系统）设置这个为true。
  </td>
</tr>
<tr>
  <td><code>spark.streaming.receiver.writeAheadLog.closeFileAfterWrite</code></td>
  <td>false</td>
  <td>
    当接收器写入一条WAL记录后是否关闭文件。当你想要使用S3（或者其他不支持flushing的文件系统）设置这个为true。
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
    <td>在cluster模式下已经提交的Spark jobs当运行失败或者重新运行时的恢复模式。
    这个只适用于运行在Standalone或者Mesos的cluster模式下。
    </td>
  </tr>
  <tr>
    <td><code>spark.deploy.zookeeper.url</code></td>
    <td>None</td>
    <td>
    当`spark.deploy.recoveryMode`被设置为ZOOKEEPER时，这个属性用来设置zookeeper的连接URL地址。
    </td>
  </tr>
  <tr>
    <td><code>spark.deploy.zookeeper.dir</code></td>
    <td>None</td>
    <td>当`spark.deploy.recoveryMode`被设置为ZOOKEEPER时，这个属性被用来设置存储恢复状态的zookeeper目录。
    </td>
  </tr>
</table>


#### Cluster Managers
每一个Spark的集群管理器都有它额外的配置选项。每一种模式的选项可以在如下页面中找到：

##### [YARN](running-on-yarn.html#configuration)

##### [Mesos](running-on-mesos.html#configuration)

##### [Standalone Mode](spark-standalone.html#cluster-launch-scripts)

# Environment Variables

某些Spark的环境变量设置可以通过变量进行配置，这些是从`conf/spark-env.sh`Spark安装目录的脚本文件中读取的(或者在WINDOWS中是`conf/spark-env.cmd`)。在Standalone和Mesos模式中，这些文件能提供针对机器的相关信息，例如主机名。在运行本地Spark任务或者提交脚本时，这个文件将会被引入。

注意`conf/spark-env.sh`文件在Spark安装后是不存在的。然而，你可以复制`conf/spark-env.sh.template`文件来创建它。确保你复制的文件是可执行的。

下面的这些变量可以在`spark-env.sh`中进行配置:


<table class="table">
  <tr><th style="width:21%">Environment Variable</th><th>Meaning</th></tr>
  <tr>
    <td><code>JAVA_HOME</code></td>
    <td>Java的安装位置。(如果它没有在你的默认<code>PATH</code>下)。</td>
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
    <td>需要绑定的该机器的IP地址。</td>
  </tr>
  <tr>
    <td><code>SPARK_PUBLIC_DNS</code></td>
    <td>你的Spark程序广播给其他主机时的主机名。</td>
  </tr>
</table>

除了上面的配置，这里还有其他的一些Spark选项可以被设置
[standalone cluster scripts](spark-standalone.html#cluster-launch-scripts)，例如CPU的数目，使用每台机器的最大的内存数。

由于`spark-env.sh`是一个shell脚本，这里面某些选项能被自动设置 -- 例如，你可以通过寻找指定网络接口来计算`SPARK_LOCAL_IP`的值。

注意：当在YARN的`cluster`模式上上运行Spark任务时，环境变量的设置需要在`conf/spark-defaults.conf`使用`spark.yarn.appMasterEnv.[EnvironmentVariableName]`的方式设置。在`spark-env.sh`设置的环境变量不会在YARN的`cluster`模式应用的Master进程中生效。参见[YARN-related Spark Properties](running-on-yarn.html#spark-properties)获取更多信息。

# Configuring Logging

Spark使用[log4j](http://logging.apache.org/log4j/)来记录日志。你可以在`conf`目录中增加配置`log4j.properties`文件来增加你自己的配置。一种方式是拷贝
`log4j.properties.template`文件来实现。

# Overriding configuration directory

为指定一个不同于"SPARK_HOME/conf"默认的的配置目录，你可以设置SPARK_CONF_DIR属性。Spark将使用这个目录中的配置文件(spark-defaults.conf, spark-env.sh, log4j.properties等等)。

# Inheriting Hadoop Cluster Configuration

你如果你计划使用Spark来读写HDFS中的数据，这里有两个Hadoop的配置文件需要被包含在Spark的 classpath:

* `hdfs-site.xml`, 提供HDFS客户端的默认行为。
* `core-site.xml`, 提供默认的文件系统名称。

这些配置文件在CDH和HDP版本里的位置是不同的，但是一个共同点都是在`/etc/hadoop/conf`目录中。在一些工具中，像Cloudera Manager，在系统启动时会创建配置文件，但是只提供了一种机制去下载它们的香醇。

为了使这些文件对Spark是可见的，`$SPARK_HOME/spark-env.sh`中设置`HADOOP_CONF_DIR`来包含这些需要的配置文件。