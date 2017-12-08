---
layout:     post
title:      "Apache Spark SQL"
subtitle:   "Catalyst Optimizer in Spark SQL"
date:       2017-12-07
author:     "xp"
header-img: "img/post-bg-catalyst.png"
header-mask: 0.3
catalog:    true
tags:
    - Spark SQL
    - Optimizer
---

> From: [Catalyst Optimizer in Spark SQL](http://www.waitingforcode.com/apache-spark-sql/catalyst-optimizer-in-spark-sql/read)

Catalyst Optimizer的技术类似于关系型数据库的查询计划和执行。第一小部分对它进行了定义。第二部分阐述了工作流和查询计划的关联关系。最后一部分展示了两个查询和它们优化的步骤。

**Catalyst Optimizer** （CO）对帮助Dataset提升执行效率操作是一个很重要的影响因素，这是一个内部的查询优化器。它把在构建在Dataset上的转换操作“翻译”成对应的物理执行计划。因此，它和用来创建RDD的物理执行计划的[DAG scheduler][1]是很相似的。

   [1]: http://www.waitingforcode.com/apache-spark/directed-acyclic-graph-in-spark/read#dag_scheduler

在性能方面，CO对Dataset来说是十分重要的。自从它能理解使用数据的结构和在它上面执行的操作，这个优化器就能执行一些决策来帮助减少执行的时间。感谢函数式编程的出现（模式匹配和quasiquotes），它使自定义执行优化成为可能。（[feature still experimental at the time of writing][2]）。

   [2]: https://github.com/apache/spark/blob/689de920056ae20fe203c2b6faf5b1462e8ea73c/sql/core/src/main/scala/org/apache/spark/sql/ExperimentalMethods.scala

从大的方面来说，CO把查询翻译成**abstract syntax tree**（AST），AST的节点就代表着执行一个查询所需要的操作。CO将会尝试应用一系列预定义的规则来优化查询节点，举例说合并3个过滤器为1个。因为树的节点是不可变的，规则应用后会创建一个新的总是经过优化后的比之前更好的一颗树。

在解释CO具体做了什么之前，我们必须先清楚我们使用的一些概念的细节：

  * **logical plan** - 一系列的代数或者语言构造，例如：在SQL中的SELECT、GROUP BY或者UNION关键词。它通常代表了一颗树由节点构成的树。
  * **physical plan** - 和逻辑计划相似，因为它也是由一颗树来表示的。它和逻辑执行计划的不同地方在于它是更低级的操作。
  * **unoptimized/optimized plans** - 未经优化的计划可以认为CO还没有对其进行任何操作。一个已经被CO处理和执行一些优化操作后的查询就变成了优化查询计划。（例如：合并filter()方法、替换一些基础参数、改善大部分的性能）。

更具体的来说，CO是在上面提到的3个列表项中工作的。它把未经优化的逻辑查询计划演变成优化后的物理查询计划，达到此目的由以下步骤组成：

  1. CO通过使用预先定义的基于规则的优化方法来优化逻辑查询计划。这些优化操作由以下这些项组成：
    * 谓词或投影下推 - 在计算过程中提前消除不符合预置条件的数据
    * 重新排列过滤器
    * 转换小数操作为长整型数操作
    * 使用Java的方法 _startsWith(String)_ 或者 _contains(String)_ 来替换某些正则表达式
    * 简化if-else从句
  2. 创建被优化后的逻辑计划。
  3. CO从优化后的逻辑计划来构建多个物理计划。一个物理计划定义了数据是如何被计算的，这个计划也是被优化过的。优化的点应关注：合并不同的filter()、在数据源层面向数据源直接发送谓词/投影下推操作来消除不相关的数据。
  4. CO决定从执行成本层面来决定哪一个物理计划被选中并被执行。CO也有一个指标的概念用来估计计划执行的成本。
  5. CO将会为最优的物理计划生成相应的字节码。这种生成技术得归功于Scala的被称作 _quasiquotes_ 的特性。这一步是由 _cost-based optimization_ 进行优化的。
  6. 一旦一个物理计划被定义，它就会被执行并且返回数据给Dataset。

### Cost-based optimization 

**Cost-based optimization** - 对给定的查询，这种优化器看起来能适应所有的可能的场景。它会给每一个这样子的场景附加一个 _cost_ 值。这个参数指明了指定场景下的执行效率。在此场景下拥有最低成本消耗的查询将会被选中和执行。
  
在RDBMS中经常会使用例如在指定key下被索引元素的数量、表中的行的数量来代表执行依赖的指标参考。
  
它和基于规则优化器的不同在于，基于规则的优化器只是简单的在语句上应用一系列优化规则。在数据集产生时，只有4个步骤是基于成本优化操作的。其他的优化操作是基于规则的。

为了了解在Spark的结构化数据操作过程中发生了什么，我们使用如下的例子：
    
    private static final SparkSession SESSION = SparkSession.builder() 
      .master("local[1]")
      .config("spark.ui.enabled", "false")
      .config("spark.eventLog.enabled", "true")
      .config("spark.eventLog.dir", "/tmp/spark")
      .appName("CatalystOptimizer Test").getOrCreate();
    
    @Test
    public void should_get_dataframe_from_database() {
      // categories as 18 entries
      Dataset<Row> dataset = getBaseDataset("categories");
    
      Dataset<Row> filteredDataset = dataset.where("LENGTH(name) > 5")
        .where("name != 'mushrooms'")
        .limit(3);
    
      // To see logical plan, filteredDataset.logicalPlan() methods can be used,
      // such as: treeString(true), asCode()
      // To see full execution details, fileteredDataset.explain(true)
      // should be called
      assertThat(filteredDataset.count()).isEqualTo(3);
    }
    
    private Dataset<Row> getBaseDataset(String dbTable) {
      // Please note that previous query won't generate real SQL query. It will only
      // check if specified column exists. It can be observed with RDBMS query logs.
      // For the case of MySQL, below query is generated:
      // SELECT * FROM meals WHERE 1=0
      // Only the action (as filteredDataset.show()) will execute the query on database.
      // It also can be checked with query logs.
      return SESSION.read()
        .format("jdbc")
        .option("url", "jdbc:mysql://localhost:3306/fooder")
        .option("driver", "com.mysql.cj.jdbc.Driver")
        .option("dbtable", dbTable)
        .option("user", "root")
        .option("password", "")
        .load();
    }
    
    

如你所见，它获取了和符合标准的最大3个categories的元素列表。代表已定义表达式的树看起来像下面的图片一样：

![][3]

   [3]: http://www.waitingforcode.com/public/images/articles/spark_sql_co.png

为了了解树的叶子是由什么组成的，Dataset的 _logicalPlan()_ 的输出信息会更有用。

另外一个有用的来帮助理解Catalyst优化操作的命令是Dataset类和调用它的 _explain(boolean)_ 。它的boolean参数用来决定是否输出详细的信息。如果这个参数设置为true，那么它的输出不仅仅包含物理计划，同时也包含所有的逻辑计划阶段（parsed、analyzed、optimized）。我们的查询的执行计划输出类似于下面这样：
    
    == Parsed Logical Plan ==
    GlobalLimit 3
    +- LocalLimit 3
       +- Filter NOT (name#1 = mushrooms)
          +- Filter (length(name#1) > 5)
             +- Relation[id#0,name#1] JDBCRelation(categories) [numPartitions=1]
    
    == Analyzed Logical Plan ==
    id: int, name: string
    GlobalLimit 3
    +- LocalLimit 3
       +- Filter NOT (name#1 = mushrooms)
          +- Filter (length(name#1) > 5)
             +- Relation[id#0,name#1] JDBCRelation(categories) [numPartitions=1]
    
    == Optimized Logical Plan ==
    GlobalLimit 3
    +- LocalLimit 3
       +- Filter ((length(name#1) > 5) && NOT (name#1 = mushrooms))
          +- Relation[id#0,name#1] JDBCRelation(categories) [numPartitions=1]
    
    == Physical Plan ==
    CollectLimit 3
    +- *Filter (length(name#1) > 5)
       +- *Scan JDBCRelation(categories) [numPartitions=1] [id#0,name#1] PushedFilters: [*Not(EqualTo(name,mushrooms))], ReadSchema: struct
    
    

如你在“Physical Plan”中所看到的那样，执行计划流是由消除了categories数据的名称列等于“mushrooms”的数据库查询组成。剩下的工作就是在Spark的层面来完成了。我们来分析一下MySQL数据库的例子，下面的这些查询应该在数据库层面被执行：
    
    2017-02-04T08:53:44.152193Z         5 Query     SET character_set_results = NULL
    2017-02-04T08:53:44.152380Z         5 Query     SET autocommit=1
    2017-02-04T08:53:44.152835Z         5 Query     SELECT `id`,`name` FROM categories WHERE ((NOT (`name` = 'mushrooms')))
    
    

我们对计划解释进行分析，我们可以看到优化后的逻辑计划把两个分开定义的 _where(String)_ 进行了组合。我为了搞清哪一个规则被应用于合并它们，我们可以去查看Spark的日志，更具体的来说，是查看包含“=== Applying Rule”的这条文本。由上所术的例子我们能找到如下的日志条目：
    
    === Applying Rule org.apache.spark.sql.catalyst.optimizer.CombineFilters ===
     GlobalLimit 3                                                                  GlobalLimit 3
     +- LocalLimit 3                                                                +- LocalLimit 3
    !   +- Filter NOT (name#1 = mushrooms)                                                +- Filter ((length(name#1) > 5) && NOT (name#1 = mushrooms))
    !      +- Filter (length(name#1) > 5)                                                 +- Relation[id#0,name#1] JDBCRelation(categories) [numPartitions=1]
    !         +- Relation[id#0,name#1] JDBCRelation(categories) [numPartitions=1]   
                     (org.apache.spark.sql.execution.SparkOptimizer:62)
    
    
    

看看我们如何分析 **the second query**，但是仅仅对物理计划的点：
    
    @Test
    public void should_get_data_from_bigger_table() {
      Dataset<Row> dataset = getBaseDataset("meals");
    
      Dataset<Row> filteredDataset = dataset.where("LENGTH(name) > 5")
        .where("name != 'mushrooms'")
        .where("name NOT LIKE 'Ser%'")
        .orderBy(new Column("name").desc())
        .limit(3);
    
      filteredDataset.show();
      assertThat(filteredDataset.count()).isEqualTo(3);
    }
    
    

由 _explain()_ 生成的输出：
    
    == Physical Plan ==
    TakeOrderedAndProject(limit=3, orderBy=[name#1 DESC NULLS LAST], output=[id#0,name#1])
    +- *Filter (length(name#1) > 5)
       +- *Scan JDBCRelation(meals) [numPartitions=1] [id#0,name#1] PushedFilters: [*Not(EqualTo(name,mushrooms)), *Not(StringStartsWith(name,Ser))], ReadSchema: struct
    
    

然而，相关的查询看起来像这样：
    
    SELECT `id`,`name` FROM meals WHERE ((NOT (`name` = 'mushrooms'))) AND ((NOT (`name` LIKE 'Ser%')))
    
    

由于LIMIT和SORT没有下推到database，这看起来有些奇怪。特别是当这个查询操作是在一个有14400行数据的表上执行的时候。反过来说，排序和结果的限定在返回的RDD里已经应用通过生成的代码了。这个也可以在日志中看到：

展示生成的排序：
    
    /* 001 */ public SpecificOrdering generate(Object[] references) {
    /* 002 */   return new SpecificOrdering(references);
    /* 003 */ }
    /* 004 */
    /* 005 */ class SpecificOrdering extends org.apache.spark.sql.catalyst.expressions.codegen.BaseOrdering {
    /* 006 */
    /* 007 */   private Object[] references;
    /* 008 */
    /* 009 */
    /* 010 */   public SpecificOrdering(Object[] references) {
    /* 011 */     this.references = references;
    /* 012 */
    /* 013 */   }
    /* 014 */
    /* 015 */
    /* 016 */
    /* 017 */   public int compare(InternalRow a, InternalRow b) {
    /* 018 */     InternalRow i = null;  // Holds current row being evaluated.
    /* 019 */
    /* 020 */     i = a;
    /* 021 */     boolean isNullA;
    /* 022 */     UTF8String primitiveA;
    /* 023 */     {
    /* 024 */
    /* 025 */       UTF8String value = i.getUTF8String(1);
    /* 026 */       isNullA = false;
    /* 027 */       primitiveA = value;
    /* 028 */     }
    /* 029 */     i = b;
    /* 030 */     boolean isNullB;
    /* 031 */     UTF8String primitiveB;
    /* 032 */     {
    /* 033 */
    /* 034 */       UTF8String value = i.getUTF8String(1);
    /* 035 */       isNullB = false;
    /* 036 */       primitiveB = value;
    /* 037 */     }
    /* 038 */     if (isNullA && isNullB) {
    /* 039 */       // Nothing
    /* 040 */     } else if (isNullA) {
    /* 041 */       return 1;
    /* 042 */     } else if (isNullB) {
    /* 043 */       return -1;
    /* 044 */     } else {
    /* 045 */       int comp = primitiveA.compare(primitiveB);
    /* 046 */       if (comp != 0) {
    /* 047 */         return -comp;
    /* 048 */       }
    /* 049 */     }
    /* 050 */
    /* 051 */     return 0;
    /* 052 */   }
    /* 053 */ }
     (org.apache.spark.sql.catalyst.expressions.codegen.CodeGenerator:58)
    
    

为了获取排序和限制后的数据，Spark使用RDD的 _takeOrdered(Integer)_ 方法。激活 _spark.eventLog.enabled_ 和 _spark.eventLog.dir_ 参数，通过调试断点或者分析JSON的事件日志可以看到相应的数据。下面这个关于生成RDD的部分已经被放置到 _SparkListenerStageCompleted_ 事件中了： 

展示SparkListenerStageCompleted事件数据
    
    {
      "Event":"SparkListenerStageCompleted",
      "Stage Info":{
        "Stage ID":0,
        "Stage Attempt ID":0,
        "Stage Name":"show at CatalystOptimizerTest:46",
        "Number of Tasks":1,
        "RDD Info":[
          {
            "RDD ID":3,
            "Name":"MapPartitionsRDD",
            "Scope":"{\"id\":\"4\",\"name\":\"takeOrdered\"}",
            "Callsite":"show at CatalystOptimizerTest:46",
            "Parent IDs":[
                2
            ],
            // ...
          },
          {
            "RDD ID":0,
            "Name":"JDBCRDD",
            "Callsite":"show at CatalystOptimizerTest:46",
            "Parent IDs":[
    
            ],
            "Storage Level":{
                "Use Disk":false,
                "Use Memory":false,
                "Deserialized":false,
                "Replication":1
            },
            "Number of Partitions":1,
            "Number of Cached Partitions":0,
            "Memory Size":0,
            "Disk Size":0
          },
          {
            "RDD ID":1,
            "Name":"MapPartitionsRDD",
            "Scope":"{\"id\":\"0\",\"name\":\"WholeStageCodegen\"}",
            "Callsite":"show at CatalystOptimizerTest:46",
            "Parent IDs":[
                0
            ],
            // ...
          },
          {
            "RDD ID":2,
            "Name":"MapPartitionsRDD",
            "Scope":"{\"id\":\"3\",\"name\":\"map\"}",
            "Callsite":"show at CatalystOptimizerTest:46",
            "Parent IDs":[
                1
            ],
            // ...
          }
        ],
        "Parent IDs":[],
        "Details":"org.apache.spark.sql.Dataset.show(Dataset.scala:604)\ncom.waitingforcode.ml.CatalystOptimizerTest.should_get_data_from_bigger_table(CatalystOptimizerTest:46)
        ...",
        "Accumulables":[
          {
      // Number of records read from database
            "ID":2,
            "Name":"number of output rows",
            "Value":"13241",
            "Internal":true,
            "Count Failed Values":true,
            "Metadata":"sql"
          },
      // Number of results after applying LENGTH(name) > 5 filter
      // at Spark level
          {
            "ID":1,
            "Name":"number of output rows",
            "Value":"13216",
            "Internal":true,
            "Count Failed Values":true,
            "Metadata":"sql"
          }, 
          // ...
        ]
      }
    }
    
    

这篇文章提供了一些在SPARK SQL模块中使用Catalyst Optimizer的内部信息。第一部分简短的定义了相关的主要信息。第二部分展示了更多的详细信息并且展示了转换工作流如何将查询优化为物理计划的。最后一部分，通过2个不同查询复杂度的例子，展示了CO的真实行为。它证述了所有如上所述 - 下推特征和动态代码生成也一样（同被证述）。