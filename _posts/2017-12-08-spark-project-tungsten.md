---
layout:     post
title:      "Apache Spark SQL"
subtitle:   "Spark Project Tungsten"
date:       2017-12-08
author:     "xp"
header-img: "img/post-bg-spark-project-tungsten.png"
header-mask: 0.3
catalog:    true
tags:
    - Spark SQL
    - Tungsten
---

> From: [Spark Project Tungsten](http://www.waitingforcode.com/apache-spark-sql/spark-project-tungsten/read)

This post presents Project Tungsten and its impact on Spark ecosystem. The first part describes a little history of it, showing the main worked points. The second part shows one of improvements introduced by this project - UnsafeRow. The last part explains where and how Spark processes data in binary format. 

## Project Tungsten history 

Project Tungsten (PT) officially [started in April 2015][1]. It was the result of analyze of many data processing issues. The finding was that the major part of bottlenecks weren't caused by initially expected I/O or networking problems but rather by CPU and memory limitations. 

   [1]: https://issues.apache.org/jira/browse/SPARK-7075

To handle these problems a set of solutions was grouped in Project Tungsten. In short, project's goals consisted on optimizing Spark jobs for CPU and memory efficiency. More concretely, it meant: 

  * new serialization format, more compact and faster than usual methods (Kryo, Java serialization), called _unsafe row_ (backed by raw memory)
  * reduced memory footprint thanks to unsafe row
  * support for on-heap and off-heap allocation; the off-heap allocation is possible thanks to unsafe row format that doesn't depend on JVM
  * faster execution of sorting and hashing for aggregation, join and shuffle operations - Spark can often do some of SQL operations as aggregation on serialized form of data
  * less time spent on waiting on fetching data from memory thanks to new _cache-friendly mechanism_. This improvement is also related to faster execution of sorting and hashing from the 4th point.
  * shuffle optimized thanks to better unsafe row throughput.

## UnsafeRow 

More compact representation and more efficient execution are achieved thanks to _org.apache.spark.sql.catalyst.expressions.UnsafeRow_. This class represents format used to serialize a Row. Such serialized row is composed of 3 regions: 

  1. Null Bit set Bitmap region - null tracking bitmap, used also to distinguish the beginning of new row
  2. Region grouping values of fixed length:
    * it takes 0 for null values
    * for values considered as fixed-length (NullType, BooleanType, ByteType,ShortType, IntegerType, LongType, FloatType,DoubleType, DateType, TimestampType - see _org.apache.spark.sql.catalyst.expressions.UnsafeRow#isFixedLength_ to be up-to-date), their real values are stored
    * for values considered as variable-length (others than mentioned previously), this part defines theirs _relative offsets_, i.e. the place where they start in section of variable length data
  3. Data section of variable length - here are placed entries having variable length. For String, the first stored word corresponds to the length and the second word to String's content bytes encoded in UTF-8.

To understand this composition, nothing better than some examples: 
    
    public static class Person {
      private String login;
      private String city;
      private int age;
      public Person(login, city, age);
      // getters/setters omitted for brevity
    }
    
    public static class Numbers {
      private int number1;
      private int number2;
      public Numbers(number1, number2);
      // getters/setters omitted for brevity
    }
    
    public static class City {
      private String name; 
      private String country; 
      private String continent;
      public City(name, country continent);
      // getters/setters omitted for brevity
    }
    
    

Below picture show how objects representing Person, Numbers, City classes are stored internally in Spark SQL as UnsafeRow: 

  * _Person("Login_Login_Login_Login", "London", 30)_
  * _Numbers(200, 333)_
  * _City("London", "the United Kingdom", "Europe")_
![Spark Project Tungsten unsafe row representation][2]

   [2]: http://www.waitingforcode.com/public/images/articles/spark_sql_tungsten.png

Did you notice something strange ? After all "London" has 6 letters, "Login_Login_Login_Login" 23. According to that, the mapping for Person object should look like: 30 - 32 - 40 - 23 - 63 - 69. But it's not the case because every word is rounded up to the nearest multiple of 8. Thus, if the word has a real length of 6, in unsafe row it's considered as 8-length word. Similarly for the word of 23 that is rounded to 24. 

Other class involved in serialization to UnsafeRow is _UnsafeRowWriter_. This object is used to write data to raw memory format. Its use can be viewed in projection of dynamically generated code (_SpecificUnsafeProjection_: 
    
    class SpecificUnsafeProjection extends org.apache.spark.sql.catalyst.expressions.UnsafeProjection {
    
      private Object[] references;
      private UnsafeRow result;
      private org.apache.spark.sql.catalyst.expressions.codegen.BufferHolder holder;
      private org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter rowWriter;
    
      public SpecificUnsafeProjection(Object[] references) {
        this.references = references;
        result = new UnsafeRow(2);
        this.holder = new org.apache.spark.sql.catalyst.expressions.codegen.BufferHolder(result, 0);
        this.rowWriter = new org.apache.spark.sql.catalyst.expressions.codegen.UnsafeRowWriter(holder, 2);
    
      }
      // ... omitted for brevity
      public UnsafeRow apply(InternalRow i) {
        rowWriter.zeroOutNullBytes(); 
        boolean isNull1 = i.isNullAt(0);
        com.waitingforcode.sql.TungenTest$Numbers value1 = isNull1 ? null : ((com.waitingforcode.sql.TungenTest$Numbers)i.get(0, null));
        boolean isNull = true;
        int value = -1;
        if (!isNull1) { 
          isNull = false;
          if (!isNull) {
            value = value1.getN1();
          }
    
        }
        if (isNull) {
          rowWriter.setNullAt(0);
        } else {
          rowWriter.write(0, value);
        }
      // ...  
    
    

## Binary data processing 

One of Tungsten's important features is the direct manipulation of serialized object (binary format) instead of deserialized Java objects. One of functions using this concept is grouping with aggregation where classes like _org.apache.spark.sql.execution.UnsafeFixedWidthAggregationMap_ or _org.apache.spark.sql.execution.UnsafeKVExternalSorter_ are involved. Below code shows some points about them: 
    
    private static final SparkSession SESSION = SparkSession.builder()
      .master("local[1]")
      .appName("Unsafe aggregation").getOrCreate();
    
    public static void main(String[] args) {
      // name, age, annual salary
      Employee employeeA = Employee.valueOf("A", 30, 30_000);
      Employee employeeB = Employee.valueOf("B", 25, 21_000);
      Employee employeeC = Employee.valueOf("C", 44, 41_000);
      Employee employeeD = Employee.valueOf("D", 39, 35_000);
      Employee employeeE = Employee.valueOf("E", 25, 35_000);
      Employee employeeF = Employee.valueOf("F", 30, 28_000);
    
      Dataset<Employee> dataset = SESSION.createDataset(asList(employeeA, employeeB, employeeC, 
        employeeD, employeeE, employeeF), Encoders.bean(Employee.class));
    
      Dataset<Row> people = dataset.groupBy("age").agg(functions.sumDistinct("age"));
      people.foreach(rdd -> {});
    } 
    
    

Generated code uses unsafe objects: 
    
    final class GeneratedIterator extends org.apache.spark.sql.execution.BufferedRowIterator {
      private Object[] references;
      private scala.collection.Iterator[] inputs;
      private boolean agg_initAgg;
      private org.apache.spark.sql.execution.aggregate.HashAggregateExec agg_plan;
      private org.apache.spark.sql.execution.UnsafeFixedWidthAggregationMap agg_hashMap;
      private org.apache.spark.sql.execution.UnsafeKVExternalSorter agg_sorter;
      private org.apache.spark.unsafe.KVIterator agg_mapIter;
      private org.apache.spark.sql.execution.metric.SQLMetric agg_peakMemory;
      private org.apache.spark.sql.execution.metric.SQLMetric agg_spillSize;
      private scala.collection.Iterator inputadapter_input;
    
      // omitted for brevity
           
      private void agg_doAggregateWithKeys() throws java.io.IOException {
        agg_hashMap = agg_plan.createHashMap();
    
        while (inputadapter_input.hasNext()) {
          InternalRow inputadapter_row = (InternalRow) inputadapter_input.next();
          int inputadapter_value = inputadapter_row.getInt(0);
          
          // a lot of code omitted for brevity
          // generate grouping key
          agg_rowWriter.zeroOutNullBytes();
    
          agg_rowWriter.write(0, inputadapter_value);
    
          long agg_value1 = (long) inputadapter_value;
          agg_rowWriter.write(1, agg_value1);
          agg_value6 = 42;
    
          agg_value6 = org.apache.spark.unsafe.hash.Murmur3_x86_32.hashInt(inputadapter_value, agg_value6);
    
          boolean agg_isNull8 = false;
          long agg_value8 = (long) inputadapter_value;
          agg_value6 = org.apache.spark.unsafe.hash.Murmur3_x86_32.hashLong(agg_value8, agg_value6);
            
          // try to get the buffer from hash map
          agg_unsafeRowAggBuffer =
          agg_hashMap.getAggregationBufferFromUnsafeRow(agg_result, agg_value6);
            
          if (agg_unsafeRowAggBuffer == null) {
            if (agg_sorter == null) {
              agg_sorter = agg_hashMap.destructAndCreateExternalSorter();
            } else {
              agg_sorter.merge(agg_hashMap.destructAndCreateExternalSorter());
            }
    
            // the hash map had be spilled, it should have enough memory now,
            // try  to allocate buffer again.
            agg_unsafeRowAggBuffer =
            agg_hashMap.getAggregationBufferFromUnsafeRow(agg_result, agg_value6);
            if (agg_unsafeRowAggBuffer == null) {
              // failed to allocate the first page
              throw new OutOfMemoryError("No enough memory for aggregation");
            }
          } 
          if (shouldStop()) return;
        }
    
        agg_mapIter = agg_plan.finishAggregate(agg_hashMap, agg_sorter, agg_peakMemory, agg_spillSize);
      }
    
      protected void processNext() throws java.io.IOException {
        if (!agg_initAgg) {
          agg_initAgg = true;
          long wholestagecodegen_beforeAgg = System.nanoTime();
          agg_doAggregateWithKeys();
          wholestagecodegen_aggTime.add((System.nanoTime() - wholestagecodegen_beforeAgg) / 1000000);
        }
    
        // output the result 
        while (agg_mapIter.next()) {
          wholestagecodegen_numOutputRows.add(1);
          UnsafeRow agg_aggKey = (UnsafeRow) agg_mapIter.getKey();
          UnsafeRow agg_aggBuffer = (UnsafeRow) agg_mapIter.getValue();
    
          int agg_value10 = agg_aggKey.getInt(0);
          long agg_value11 = agg_aggKey.getLong(1);
          agg_rowWriter1.write(0, agg_value10);
    
          agg_rowWriter1.write(1, agg_value11);
          append(agg_result1);
    
          if (shouldStop()) return;
        }
    
        agg_mapIter.close();
        if (agg_sorter == null) {
          agg_hashMap.free();
        }
      }
    } 
    
    

UnsafeFixedWidthAggregationMap frequently called method is _getAggregationBufferFromUnsafeRow(UnsafeRow, int)_ generating aggregation buffer (UnsafeRow) for defined UnsafeRow and hash. Internally it uses _BytesToBytesMap_ that is a structure helping to aggregate rows with smaller memory footprint. This class is in fact append-only hash map where keys and values are contiguous regions of bytes represented as in below image: 

![][3]

   [3]: http://www.waitingforcode.com/public/images/articles/spark_tungsten_unsafehashmap.png

All operations in UnsafeFixedWidthAggregationMap are done with UnsafeRow, i.e. data is never converted to Java object. In additional, data manipulated by this class is consistent with other tuned operations, as sorting. It means that _UnsafeExternalSorter_ can operate on the same entries as aggregator. 

This post covers some main information about Project Tungsten. The first part lists improvements brought by it. It shows that they're especially related to new storage format based on raw memory, able to be stored on- and off-heap. The second part presents some details about this new format while the last part shows how aggregation operation uses works directly on this format instead of deserialized one. 
