---
layout:     post
title:      "MySQL INFORMATION_SCHEMA使用"
subtitle:   "表锁、表空间、死锁、连接信息等查看"
date:       2017-02-07
author:     "xp"
header-img: "img/post-bg-os-metro.jpg"
header-mask: 0.3
catalog:    true
tags:
    - MySQL
    - Tool
---

## 查看指定库中表的数据量和数据大小信息
```sql
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_ROWS, DATA_LENGTH + INDEX_LENGTH AS all_length
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'market_compass_v2'
-- AND TABLE_NAME = 'model_offline_year_weekly'
ORDER BY table_rows DESC;
```

下面将逐一说明参数的用法及使用场景。

## 通用参数说明
- **-h,--help**：打印命令使用帮助详细信息。
- **-f,--sql-file**：需要执行的Hive SQL文件。
- **-T,--export-type**：SQL执行结果的导出类型，默认为***jdbc***，另支持***none***、***csv***、***hive***模式，后续将对这几种导出模式作说明。
- **-p,--sql-param**：需要传入SQL文件的参数，例如`--sql-param execDate=20160701`，若有多个SQL参数需要传入，则多次使用`--sql-param`参数，`注意`：不能使用工具的关键字，例如`table`等。


## jdbc模式参数说明
此模式为默认执行模式，在未指定`-T,--export-type`参数或参数的值为`jdbc`的情况下将对SQL文件最后一行的执行结果导入指定的数据库中。

- **--export-config-file**：指定导入数据库的默认连接信息的文件名，此文件中的连接信息将被以下命令行指定的同名信息所覆盖。
- **-C,--connect-uri**：导入数据库的JDBC连接信息，例如：`--connect-uri "jdbc:mysql://192.168.100.200:3306/test?useUnicode=true&characterEncoding=utf-8"`。
- **-t,--table**：需要导入数据的表名。
- **--insert-sql**：自定义使用的插入SQL
- **-u,--username**：数据库用户名。
- **-P,--password**：数据库密码。
- **-v,--verbose**：是否打印SQL详细执行结果信息。
- **--update-sql**：自定义使用的更新SQL
- **-k,--update-key**：指定导入表的主键，多列以`,`分隔，若指定，则会删除已经存在相同主键的数据记录然后导入数据，否则直接将数据导入至指定表中。
- **--delete-key**：指定删除表数据的主键，多列以`,`分隔，若指定，则以删除键删除匹配的数据记录，否则删除键的值为`--update-key`的值。
- **-m,--update-mode**：支持`updateonly`和`allowinsert`两种模式，默认为`allowinsert`模式，区别在于`allowinsert`模式会更新和插入新数据，而`updateonly`只会更新具有相同主键的数据。
- **-c,--columns**：指定数据库需要插入的列名称，以英文逗号（`,`）进行分隔，即在生成的Insert语句将指定插入的列名，若未指定，则插入指定表的所有列。
- **-b,--batch**：是否以批量模式还是单条插入模式导入数据，详见JDBC的BATCH操作。
- **-s,--batch-size**：批量模式下批量插入数据的条数。


## csv模式参数说明
此模式为将SQL执行的结果导出为指定的CSV文件。

- **--csv-include-header**：导出CSV文件是否需要包含查询的列头，取值为`true`或者`false`。
- **--csv-field-delimiter**：导出CSV文件列的分隔符，默认为`,`。
- **--csv-save-path**：导出CSV文件保存的路径。


## hive/none模式参数说明
目前为止，这两种模式在效果上是一致的，即只执行SQL文件操作，不执行后续任何操作。

## 使用示例
```bash
spark-submit \
--master spark://nn.avcdata.com:7077 \
--name sparkAppName \
--driver-memory 2G \
--executor-memory 4G \
--driver-cores 2 \
--executor-cores 4 \
--num-executors 4 \
--total-executor-cores 24 \
--driver-java-options "-Dlog4j.configuration=classpath:log4j.properties" \
--class com.avcdata.etl.launcher.Spark4HiveQLExecutor \
etl-launcher-0.0.1-jar-with-dependencies.jar \
--sql-file hdfs:///user/hue/shichangluopanAPP/sql/avc_marketcompass_brandcompetition.hql \
--connect-uri 'jdbc:mysql://192.168.100.200:3306/AVCData?useUnicode=true&characterEncoding=utf-8' \
--username dbUser \
--password 'password' \
--table avc_marketcompass_brandcompetition \
--update-key 'category,brand,platform,startweek,endweek' \
--sql-param 'execDate=20160604' \
--sql-param 'yearOffset=0' \
--sql-param 'monthOffset=0' \
--sql-param 'dayOffset=0' \
--sql-param 'relativeStartWeekOffset=-7' \
--sql-param 'relativeEndWeekOffset=-7'
```

## 其他
后续会根据项目实际情况需求，是否增加Redis通用导出模式。
