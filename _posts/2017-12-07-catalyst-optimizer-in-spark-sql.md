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

This technique is similar to relational database query planning and is executed by Catalyst Optimizer. The first short part defines it. The second part explains the workflow related to query plan. The last part shows 2 queries and the steps of their optimization. 

An important element helping Dataset to perform better is **Catalyst Optimizer** (CO), an internal query optimizer. It "translates" transformations used to build the Dataset to physical plan of execution. Thus, it's similar to [DAG scheduler][1] used to create physical plan of execution of RDD. 

   [1]: http://www.waitingforcode.com/apache-spark/directed-acyclic-graph-in-spark/read#dag_scheduler

CO is precious to Dataset in terms of performance. Since it understands the structure of used data and operations made on it, the optimizer can make some decisions helping to reduce time execution. Thanks to functional programming constructions (pattern matching and quasiquotes), is open to custom optimizations ([feature still experimental at the time of writing][2]). 

   [2]: https://github.com/apache/spark/blob/689de920056ae20fe203c2b6faf5b1462e8ea73c/sql/core/src/main/scala/org/apache/spark/sql/ExperimentalMethods.scala

From the big picture perspective, CO translates query to **abstract syntax tree** (AST) that nodes are represented by operations made to executed a query. CO will try to optimize them by applying a set of predefined rules, as for example combining 3 filters into a single one. Because tree nodes are immutable, rules application creates a new tree, being always more optimized than the previous one. 

Before explaining what CO exactly does, some of concepts it uses must be detailed before: 

  * **logical plan** - series of algebraic or language constructs, as for example: SELECT, GROUP BY or UNION keywords in SQL. It's usually represented as a tree where nodes are the constructs.
  * **physical plan** - similar to logical because also represented as a tree. But the difference is that physical plan concerns low level operations.
  * **unoptimized/optimized plans** - a plan is considered as unoptimized when CO hasn't worked on it yet. The plan becomes optimized when CO passed on it and made some optimizations (e.g.: merging filter() methods, replacing some instructions by another ones, most performant).

More exactly, CO works on 3 items listed before. It helps to move from unoptimized logical query plan to optimized physical plan, achieving that in below steps: 

  1. CO tries to optimize logical query plan through predefined rule-based optimizations. The optimization can consists on:
    * predicate or projection pushdown - helps to eliminate data not respecting preconditions earlier in the computation.
    * rearrange filter
    * conversion of decimals operations to long integer operations
    * replacement of some RegEx expressions by Java's methods _startsWith(String)_ or _contains(String)_
    * if-else clauses simplification
  2. Optimized logical plan is created.
  3. CO constructs multiple physical plans from optimized logical plan. A physical plan defines how data will be computed. The plans are also optimized. The optimization can concern: merging different filter(), sending predicate/projection pushdown directly to datasource to eliminate some data at data source level.
  4. CO determines which physical plan has the lowest cost of execution and choses it as the physical plan used for the computation. CO also has a concept of metrics used to estimate the cost of plans.
  5. CO generates bytecode for the best physical plan. The generation is made thanks to Scala's feature called _quasiquotes_. This step is optimized by _cost-based optimization _
  6. Once a physical plan is defined, it's executed and retrieved data is put to Dataset.

### Cost-based optimization 

**Cost-based optimization** - the optimizer looks at all possible scenarios to execute given query. It assigns a _cost_ for each of these scenarios. This parameter indicates the efficiency of given scenario. The scenario having the lowest cost is further chosen and executed.  
  
In RDBMS it often uses metrics which can represent for example: the number of indexed elements under given key or simply the number of rows in a table.  
  
It's different from rule-based optimization which simply applies a set of rules to statement. In Dataset generation, only the 4th step is cost-based optimization. The others optimized steps are rule-based. 

To see what happens when operations on structured data is made in Spark, below example is used: 
    
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
    
    

As you can see, it gets a list of maximally 3 categories corresponding to defined criteria. The tree representing such defined expression looks like in below picture: 

![][3]

   [3]: http://www.waitingforcode.com/public/images/articles/spark_sql_co.png

To see what leaves compose the tree, Dataset's _logicalPlan()_ proves to be useful. 

Another useful command helping to understand Catalyst's optimisations also comes from Dataset class and is called _explain(boolean)_. Its boolean parameter determines the verbosity of the output. If it's set to true, the output will contain not only physical plan, but also all phases of logical plans (parsed, analyzed and optimized). The output for our query looks like: 
    
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
    
    

As you can see in "Physical Plan" part, execution workflow is composed of database query eliminating categories those names is equal to "mushrooms". The rest of work is done on Spark's level. To see that it's true, we can enable log queries. For the case of analyzed MySQL database, below query should be executed at database's level: 
    
    2017-02-04T08:53:44.152193Z         5 Query     SET character_set_results = NULL
    2017-02-04T08:53:44.152380Z         5 Query     SET autocommit=1
    2017-02-04T08:53:44.152835Z         5 Query     SELECT `id`,`name` FROM categories WHERE ((NOT (`name` = 'mushrooms')))
    
    

At analyze of plans explanation, we can see that optimized logical plan combined two filters defined separetly in 2 _where(String)_ methods. To detect which rule was applied to merge them we can take a look at Spark's logs, and more specifically, at entries containing "=== Applying Rule" text. For described example we can find below entry: 
    
    === Applying Rule org.apache.spark.sql.catalyst.optimizer.CombineFilters ===
     GlobalLimit 3                                                                  GlobalLimit 3
     +- LocalLimit 3                                                                +- LocalLimit 3
    !   +- Filter NOT (name#1 = mushrooms)                                                +- Filter ((length(name#1) > 5) && NOT (name#1 = mushrooms))
    !      +- Filter (length(name#1) > 5)                                                 +- Relation[id#0,name#1] JDBCRelation(categories) [numPartitions=1]
    !         +- Relation[id#0,name#1] JDBCRelation(categories) [numPartitions=1]   
                     (org.apache.spark.sql.execution.SparkOptimizer:62)
    
    
    

Let's how analyze **the second query**, but only from the point of physical plan: 
    
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
    
    

The output generated by _explain()_ is: 
    
    == Physical Plan ==
    TakeOrderedAndProject(limit=3, orderBy=[name#1 DESC NULLS LAST], output=[id#0,name#1])
    +- *Filter (length(name#1) > 5)
       +- *Scan JDBCRelation(meals) [numPartitions=1] [id#0,name#1] PushedFilters: [*Not(EqualTo(name,mushrooms)), *Not(StringStartsWith(name,Ser))], ReadSchema: struct
    
    

Thus, corresponding query looks like: 
    
    SELECT `id`,`name` FROM meals WHERE ((NOT (`name` = 'mushrooms'))) AND ((NOT (`name` LIKE 'Ser%')))
    
    

It seems a little bit strange that LIMIT and SORT weren't pushed down to database. Especialy when the query operates on a table with 14400 rows. Instead, ordering and results limit are done through generated code applied on retrieved RDDs. It's also visible in logs: 

show generated ordering
    
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
    
    

To get data ordered and limited, Spark uses RDD _takeOrdered(Integer)_ method. It can be seen by adding debugging breakpoints or by analyzing JSON event log, activated by _spark.eventLog.enabled_ and _spark.eventLog.dir_ properties. The part about generated RDDs is placed inside _SparkListenerStageCompleted_ event: 

show SparkListenerStageCompleted event data
    
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
    
    

This post provides a little insight on Spark Catalyst Optimizer used in Spark SQL module. The first part defines shortly the main information about it. The second part goes more into details and shows the workflow of transforming parsed query to optimized physical plan. The last part, through 2 examples of different query complexity, shows how CO really behaves. It proves all that was described previously - pushdown features as well as dynamic generation of code. 
