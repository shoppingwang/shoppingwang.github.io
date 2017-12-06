---
layout:     post
title:      "Apache Spark SQL"
subtitle:   "Dataset in Spark SQL"
date:       2017-12-06
author:     "xp"
header-img: "img/post-bg-theory.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Spark SQL
---

> From: [Dataset in Spark SQL](http://www.waitingforcode.com/apache-spark-sql/dataset-in-spark-sql/read)

本文章专注于Dataset的讲解。第一部分定义了Dataset在结构化和半结构化数据的使用抽象。第二部分展示了Dataset的演化历程并给出它和RDD在内部的联系。最后一部分展示了如何去从各种各样的源去创建Dataset的示例代码。

## Dataset definition 

Spark的 _org.apache.spark.sql.Dataset_ 有如下特征要点：

  * **Encoder and QueryExecution tuple** - 在Dataset的内部，它是由 _org.apache.spark.sql.execution.QueryExecution_ 和 _org.apache.spark.sql.Encoder_ 这两个tuple构成的。第一个对象定义了相关查询执行的工作流。Encoder的职责是在JVM对象和Spark SQL内部对象（row）进行转换。
  * **encoders** - 和RDD不同，Dataset没有使用Java或者Kryo的序列化的字节来代表JVM的对象。它使用已经存在被引用的Encoder来实现它的对象格式。这种格式更加简洁并且允许Spark在没有必要反序列化前执行很多的操作（filtering、sorting、hashing）。和其他许多基准测试一样（例如[Structuring Apache Spark 2.0: SQL, DataFrames, Datasets And Streaming - by Michael Armbrust][1]），它提高了Spark的性能。
  * **QueryExecution** - Dataset在内部是由一种类似催化剂的逻辑计划 _QueryExecution_ 来表示的。这种计划代表了的实际会在Dataset中产生数据的查询。
  * **schema-based** - 和RDD不同，Dataset是结构化的，例如，在它持有的数据上定义了一个schema。
  * **strongly-typed** - Dataset是强类型的。它能帮助你在结构化和半结构化数据上正确的工作。
  * **immutable and distributed** - 和RDD一样，Dataset只能通过transformation来构建一个新的Dataset实例来达到修改的目的。
  * **lazy evaluated** - 和RDD类似，Dataset也是延迟计算的，例如，只有在它触发一个action操作后才会触发相应的计算。
  * **optimized** - Dataset是被一个叫做Catalyst Optimizer优化器进行查询优化的。（这个概念在[Catalyst Optymizer in Spark][2]这篇文章中有简述）。
  * **uniform and universal** - Dataset能处理来自各种各样的源，从自然的支持关系型数据库开始，到JSON或者HDFS文件，结束于NoSQL类似ElasticSearch或者Cassandra这样的NoSQL引擎（它们的连接可以使用第三方的插件来处理）。
  * **RDD interoperatbility** - Dataset能够很容易的被转换为RDD，这得归功于 _rdd()_ 或者 _toJavaRDD()_ 这两个方法。

   [1]: https://youtu.be/1a4pgYzeFwE?t=1111
   [2]: http://www.waitingforcode.com/apache-spark-sql/catalyst-optimizer-in-spark-sql/read

## Dataset and RDD relationship 

历史上的Dataset是由 _SchemaRDD_ 演化而来的。SchemaRDD是 _RDD_ 类型化 _Row_ 了的对象的扩展。如它的名字所展示的一样，可以把它想象成一个结构化的RDD（带有schema的RDD）。在我们分析它的API（例如在[commit deleting SchemaRDD from Spark][3]中）的时候，我们能很容易的发现它成功的原因是因为它使用一些简单的概念。它关联了 _LogicalPlan_ 类，实际上是 _QueryExecution_ 的一部分。相同的QueryExecution依次组成Dataset。

   [3]: https://github.com/apache/spark/commit/119f45d61d7b48d376cca05e1b4f0c7fcf65bfa8#diff-1b97e54687301e5840bb97e576f83ee6

与此同时，Spark 1.3在API上带来了一些改变。特别是在[SPARK-5097][4]中。然后SchemaRDD就被 _DataFrame_ 所替代了。重新设计的API，DataFrame不在和RDD之间有任何的耦合关系。它是一个简单的对象，由一个SQLContext和上面已经提及的QueryExecution所组成。RDD变成了DataFrame中一个延迟计算的成员。

   [4]: https://issues.apache.org/jira/browse/SPARK-5097

Spark 1.6介绍了Dataset并且在下个版本中（2.0）DataFrame被转换成了 _Dataset<Row>_ 的别名。Dataset除了一些在API上的改变，还带来了另外一个重要的提升 - _org.apache.spark.sql.Encoder_ ，Spark有效的内部格式，能有效的提升性能。

即使Dataset和RDDs之间只有很少的耦合，但它还是使用RDD来执行相应的计算。在Spark SQL中物理执行操作使用 _org.apache.spark.sql.execution.SparkPlan_ 来实现。负责真正执行的方法会显示的调用RDD进行相关的操作：
    
     /**
      * Returns the result of this query as an RDD[InternalRow] by delegating to `doExecute` after
      * preparations.
      *
      * Concrete implementations of SparkPlan should override `doExecute`.
      */
    final def execute(): RDD[InternalRow] = executeQuery {
      doExecute()
    }
    
    

RDD的调用过程可以通过配置 _spark.eventLog.enabled_ 和 _spark.eventLog.dir_ 这两个参数监听事件来检测到。下面是测试的代码和生成的输出：
    
    @Test
    public void should_create_dataset_from_objects() throws InterruptedException {
      City london = valueOf("London", "England", "Europe");
      City paris = valueOf("Paris", "France", "Europe");
      City warsaw = valueOf("Warsaw", "Poland", "Europe");
      City tokio = valueOf("Tokio", "Japan", "Asia");
    
      Encoder<City> cityEncoder = Encoders.bean(City.class);
      Dataset<City> dataset = SESSION.createDataset(Arrays.asList(london, paris, warsaw, tokio), cityEncoder);
    
      assertThat(dataset.count()).isEqualTo(4);
    }
    
    

输出：
    
    {
      "Event":"SparkListenerStageCompleted",
      "Stage Info":{
        "Stage Name":"count at DatasetTest.java:144",
        "Number of Tasks":1,
        "RDD Info":[
          {
            "RDD ID":3,
            "Name":"MapPartitionsRDD",
            "Scope":"{\"id\":\"3\",\"name\":\"Exchange\"}",
            "Callsite":"count at DatasetTest.java:144",
            "Parent IDs":[
                2
            ], //...
          },
          {
            "RDD ID":1,
            "Name":"MapPartitionsRDD",
            "Scope":"{\"id\":\"7\",\"name\":\"LocalTableScan\"}",
            "Callsite":"count at DatasetTest.java:144",
            "Parent IDs":[
                0
            ], //...
          },
          {
            "RDD ID":2,
            "Name":"MapPartitionsRDD",
            "Scope":"{\"id\":\"4\",\"name\":\"WholeStageCodegen\"}",
            "Callsite":"count at DatasetTest.java:144",
            "Parent IDs":[
                1
            ], //...
          },
          {
            "RDD ID":0,
            "Name":"ParallelCollectionRDD",
            "Scope":"{\"id\":\"7\",\"name\":\"LocalTableScan\"}"
            // ...
          }
        ]
        //...
      }
    }
    
    

如上所示，在调用Dataset的 _count()_ 方法后会触发底层一系列的RDD的创建。

## Dataset examples 

下面这些代码展示了如何去创建Dataset：
    
    private static final SparkSession SESSION = SparkSession.builder() 
        .master("local[1]").appName("Dataset Test").getOrCreate();
    
    
    @Test
    public void should_construct_dataset_from_json() {
      // cities.json is not correctly formatted JSON. It's because
      // .json(String) reads objects line per line which is shown
      // in should_construct_bad_dataset_from_correctly_formatted_json test
      Dataset<Row> cities = readCitiesFromJson();
    
      assertThat(cities.collectAsList()).hasSize(4);
    }
    
    @Test
    public void should_construct_bad_dataset_from_correctly_formatted_json() {
      String filePath = getClass().getClassLoader().getResource("dataset/cities_correct_format.json").getPath();
    
      Dataset<Row> cities = SESSION.read().json(filePath);
    
      // It's 8 because .json(String) takes 1 row per line, even if the row doesn't
      // contain defined object (city)
      assertThat(cities.collectAsList()).hasSize(8);
      cities.printSchema();
    }
    
    @Test
    public void should_create_dataset_from_rdd() {
      JavaSparkContext closeableContext = new JavaSparkContext(SESSION.sparkContext());
      try {
        List<String> textList = Lists.newArrayList("Txt1", "Txt2");
        JavaRDD<Row> textRdd = closeableContext.parallelize(textList)
          .map((Function<String, Row>) record -> RowFactory.create(record));
    
        StructType schema = DataTypes.createStructType(new StructField[] {
          DataTypes.createStructField("content", DataTypes.StringType, true)
        });
        Dataset<Row> datasetFromRdd = SESSION.createDataFrame(textRdd, schema);
    
        assertThat(datasetFromRdd.count()).isEqualTo(2L);
        List<Row> rows = datasetFromRdd.collectAsList();
        assertThat(rows.get(0).mkString()).isEqualTo(textList.get(0));
        assertThat(rows.get(1).mkString()).isEqualTo(textList.get(1));
      } finally {
        closeableContext.close();
      }
    }
    
    @Test
    public void should_fail_on_creating_create_dataset_from_rdd_with_incorrect_schema() {
      JavaSparkContext closeableContext = new JavaSparkContext(SESSION.sparkContext());
      try {
        List<String> textList = Lists.newArrayList("Txt1", "Txt2");
        JavaRDD<Row> textRdd = closeableContext.parallelize(textList)
          .map((Function<String, Row>) record -> RowFactory.create(record));
    
        StructType schema = DataTypes.createStructType(new StructField[]{
          DataTypes.createStructField("content", DataTypes.TimestampType, true)
        });
        Dataset<Row> datasetFromRdd = SESSION.createDataFrame(textRdd, schema);
    
        // Trigger action to make Dataset really created
        assertThatExceptionOfType(Exception.class).isThrownBy(() -> datasetFromRdd.show())
          .withMessageContaining("java.lang.String is not a valid external type for schema of timestamp");
      } finally {
        closeableContext.close();
      }
    }
    
    @Test
    public void should_print_dataset_schema() {
      Dataset<Row> cities = readCitiesFromJson();
    
      cities.printSchema();
      // Schema should look like:
      // root
      // |-- continent: string (nullable = true)
      // |-- country: string (nullable = true)
      // |-- name: string (nullable = true)
      // Here also it's difficult to test, so check simply if Dataset
      // was constructed
    
      assertThat(cities).isNotNull();
    }
    
    @Test
    public void should_get_rdd_from_dataset() {
      Dataset<Row> cities = readCitiesFromJson();
    
      // Dataset stores collected data as RDD
      // which is a Dataset's field access in Java from rdd()
      // method
      RDD<Row> scalaRdd = cities.rdd();
    
      assertThat(scalaRdd).isNotNull();
      String rddDag = scalaRdd.toDebugString();
      assertThat(rddDag).contains("FileScanRDD[3]", 
          "MapPartitionsRDD[4]", "MapPartitionsRDD[5]", "MapPartitionsRDD[6]");
      // We could also use cities.toJavaRDD()
      JavaRDD<Row> javaRdd = scalaRdd.toJavaRDD();
      List<City> citiesList = javaRdd.map(row -> fromRow(row)).collect();
      assertThat(citiesList).hasSize(4);
      assertThat(citiesList).extracting("name").containsOnly("Paris", 
        "Warsaw", "Rome", "Buenos Aires");
    }
    
    @Test
    public void should_create_dataset_from_objects() throws InterruptedException {
        City london = valueOf("London", "England", "Europe");
        City paris = valueOf("Paris", "France", "Europe");
        City warsaw = valueOf("Warsaw", "Poland", "Europe");
        City tokio = valueOf("Tokio", "Japan", "Asia");
    
        Encoder<City> cityEncoder = Encoders.bean(City.class);
        Dataset<City> dataset = 
            SESSION.createDataset(Arrays.asList(london, paris, warsaw, tokio), cityEncoder);
    
        assertThat(dataset.count()).isEqualTo(4);
    }
    
    @Test
    public void should_get_dataset_from_database() {
        InMemoryDatabase.init();
    
        Dataset<Row> dataset = SESSION.read()
          .format("jdbc")
          .option("url", InMemoryDatabase.DB_CONNECTION)
          .option("driver", InMemoryDatabase.DB_DRIVER)
          .option("dbtable", "city")
          .option("user", InMemoryDatabase.DB_USER)
          .option("password", InMemoryDatabase.DB_PASSWORD)
          .load();
    
        assertThat(dataset.count()).isEqualTo(7);
    }
    
    @Test
    public void should_show_how_physical_plan_is_planed() {
      // Datasets are strictly related to Catalyst Optimizer described
      // in another post
      Dataset<Row> cities = readCitiesFromJson();
    
      Dataset<Row> europeanCities = cities.filter((Row cityRow) -> cityRow.<String>getAs("continent").equals("Europe"))
        .orderBy(new Column("name").asc())
        .filter((Row cityRow) -> cityRow.<String>getAs("name").length() > 2)
        .limit(1);
    
      europeanCities.explain();
      // It's quite impossible to test physical plan explanation
      // It's the reason why we test the result and give physical plan output as
      // a comment:
      // == Physical Plan ==
      // CollectLimit 1
      // +- *Filter com.waitingforcode.sql.DatasetTest$$Lambda$10/295937119@69796bd0.call
      //    +- *Sort [name#2 ASC NULLS FIRST], true, 0
      //       +- Exchange rangepartitioning(name#2 ASC NULLS FIRST, 200)
      //          +- *Filter com.waitingforcode.sql.DatasetTest$$Lambda$9/907858780@69ed5ea2.call
      //             +- *FileScan json [continent#0,country#1,name#2] Batched: false, Format: JSON, Location:
      //                 InMemoryFileIndex[file:sandbox/spark/target/test-classes/dataset..., PartitionFilters:
      //                 [], PushedFilters: [], ReadSchema: struct<continent:string,country:string,name:string>
      // As you can see, the physical plan started with file reading, without any
      // pushdown predicate. After that both filters were applied
      // together.
      assertThat(europeanCities.count()).isEqualTo(1L);
    }
    
    @Test
    public void should_show_logical_and_physical_plan_phases() {
      Dataset<Row> cities = readCitiesFromJson();
    
      Dataset<Row> europeanCities = cities.filter((Row cityRow) -> cityRow.<String>getAs("continent").equals("Europe"))
        .orderBy(new Column("name").asc())
        .filter((Row cityRow) -> cityRow.<String>getAs("name").length() > 2)
        .limit(1);
    
      boolean withLogicalPlan = true;
      europeanCities.explain(withLogicalPlan);
    
      // As should_show_how_physical_plan_is_planed, it's difficult to test printed output
      // It's the reason why the output is printed here and the test concerns returned elements
      // == Parsed Logical Plan ==
      // GlobalLimit 1
      // +- LocalLimit 1
      //   +- TypedFilter com.waitingforcode.sql.DatasetTest$$Lambda$10/295937119@69796bd0,
      //       interface org.apache.spark.sql.Row, [StructField(continent,StringType,true),
      //       StructField(country,StringType,true), StructField(name,StringType,true)],
      //       createexternalrow(continent#0.toString, country#1.toString, name#2.toString,
      //       StructField(continent,StringType,true), StructField(country,StringType,true),
      //       StructField(name,StringType,true))
      //      +- Sort [name#2 ASC NULLS FIRST], true
      //        +- TypedFilter com.waitingforcode.sql.DatasetTest$$Lambda$9/907858780@69ed5ea2,
      //           interface org.apache.spark.sql.Row, [StructField(continent,StringType,true),
      //           StructField(country,StringType,true), StructField(name,StringType,true)],
      //           createexternalrow(continent#0.toString, country#1.toString, name#2.toString,
      //           StructField(continent,StringType,true), StructField(country,StringType,true),
      //           StructField(name,StringType,true))
      //           +- Relation[continent#0,country#1,name#2] json
    
      // == Analyzed Logical Plan ==
      // continent: string, country: string, name: string
      // GlobalLimit 1
      // +- LocalLimit 1
      //   +- TypedFilter com.waitingforcode.sql.DatasetTest$$Lambda$10/295937119@69796bd0,
      //       interface org.apache.spark.sql.Row, [StructField(continent,StringType,true),
      //       StructField(country,StringType,true), StructField(name,StringType,true)],
      //       createexternalrow(continent#0.toString, country#1.toString, name#2.toString,
      //       StructField(continent,StringType,true), StructField(country,StringType,true),
      //       StructField(name,StringType,true))
      //     +- Sort [name#2 ASC NULLS FIRST], true
      //       +- TypedFilter com.waitingforcode.sql.DatasetTest$$Lambda$9/907858780@69ed5ea2,
      //          interface org.apache.spark.sql.Row, [StructField(continent,StringType,true),
      //          StructField(country,StringType,true), StructField(name,StringType,true)],
      //          createexternalrow(continent#0.toString, country#1.toString, name#2.toString,
      //          StructField(continent,StringType,true), StructField(country,StringType,true),
      //          StructField(name,StringType,true))
      //         +- Relation[continent#0,country#1,name#2] json
      // Analyzed logical plan is a plan with resolved relations and attributes.
      // The resolution is made with analyzer. It works only on still unresolved
      // attributes and relations. Unlike parsed plan, we can see that row attributes
      // (continent, country, name) are defined and type.
    
      // == Optimized Logical Plan ==
      // GlobalLimit 1
      // +- LocalLimit 1
      //   +- TypedFilter com.waitingforcode.sql.DatasetTest$$Lambda$10/295937119@69796bd0, interface org.apache.spark.sql.Row, [StructField(continent,StringType,true), StructField(country,StringType,true), StructField(name,StringType,true)], createexternalrow(continent#0.toString, country#1.toString, name#2.toString, StructField(continent,StringType,true), StructField(country,StringType,true), StructField(name,StringType,true))
      //     +- Sort [name#2 ASC NULLS FIRST], true
      //       +- TypedFilter com.waitingforcode.sql.DatasetTest$$Lambda$9/907858780@69ed5ea2, interface org.apache.spark.sql.Row, [StructField(continent,StringType,true), StructField(country,StringType,true), StructField(name,StringType,true)], createexternalrow(continent#0.toString, country#1.toString, name#2.toString, StructField(continent,StringType,true), StructField(country,StringType,true), StructField(name,StringType,true))
      //         +- Relation[continent#0,country#1,name#2] json
      // Represents optimized logical plan, ie. a logical plan
      // on which rule-based optimizations were applied
    
      // == Physical Plan ==
      // ...
      // And finally physical plan is the same as in should_show_how_physical_plan_is_planed
      // It's omitted here for brevity
      assertThat(europeanCities.count()).isEqualTo(1L);
    }
    
    private Dataset<Row> readCitiesFromJson() {
      String filePath = getClass().getClassLoader().getResource("dataset/cities.json").getPath();
      return SESSION.read().json(filePath);
    }
    
    public static class City implements Serializable {
      private String name;
    
      private String country;
    
      private String continent;
    
      public String getName() {
        return name;
      }
    
      public void setName(String name) {
        this.name = name;
      }
    
      public String getCountry() {
        return country;
      }
    
      public void setCountry(String country) {
        this.country = country;
      }
    
      public String getContinent() {
        return continent;
      }
    
      public void setContinent(String continent) {
        this.continent = continent;
      }
    
      public static City fromRow(Row row) {
        City city = new City();
        city.name = row.getAs("name");
        city.country = row.getAs("country");
        city.continent = row.getAs("continent");
        return city;
      }
    
      public static City valueOf(String name, String country, String continent) {
        City city = new City();
        city.name = name;
        city.country = country;
        city.continent = continent;
        return city;
      } 
    }
    
    

这篇文章描述了Dataset在结构化和半结构化数据的处理。第一部分列出并阐述了Dataset对象的主要特征。第二部分给我们从内部视角展示了Dataset现在的格式是如何从SchemaRDD演化而来的。最后一部分内容展示了如何以不同的方式来创建Dataset（关系型数据库、JSON文件、普通对象）。
