---
layout:     post
title:      "AVCData通用HiveQL执行工具使用说明"
subtitle:   "如何实现Spark SQL数据抽取配置化"
date:       2016-07-04
author:     "xp"
header-img: "img/home-bg-o.jpg"
header-mask: 0.3
catalog:    true
tags:
    - AVCData
    - SparkSQL
    - Hive
    - Tool
---

> 此工具主要实现了通过执行[Hive SQL](http://hive.apache.org)，将SQL执行后的结果导入至支持JDBC的RDBMS中。


## 支持的命令行参数及格式
```
usage: Spark4HiveQLExecutor
 -b,--batch <batch>                           Use batch mode for
                                              underlying statement
                                              execution.
 -C,--connect-uri <jdbc-uri>                  Specify JDBC connect string.
 -c,--columns <col,col,col…>                  Columns to export to table.
    --csv-field-delimiter <field-delimiter>   Csv field delimiter.
    --csv-include-header <true or false>      If the csv file includes
                                              header line.
    --csv-save-path <save-path>               Csv file save path.
    --delete-key <col-name>                   Anchor column to use for
                                              deletes. Default to
                                              `update-key` Use a comma
                                              separated list of columns if
                                              there are more than one
                                              column.
    --export-config-file <filename>           JDBC export config file from
                                              hdfs.
 -f,--sql-file <filename>                     SQL file from hdfs.
 -h,--help                                    Lists short help.
    --insert-sql <insert-sql>                 The self-define SQL for
                                              insert operation.
 -k,--update-key <col-name>                   Anchor column to use for
                                              updates. Use a comma
                                              separated list of columns if
                                              there are more than one
                                              column.
 -m,--update-mode <mode>                      Specify how updates are
                                              performed when new rows are
                                              found with non-matching keys
                                              in database.
                                              Legal values for mode
                                              include updateonly and
                                              allowinsert (default).
 -P,--password <password>                     Set authentication password.
 -p,--sql-param <name=value>                  Sql params for execute.
 -s,--batch-size <batch-size>                 SQL batch size for table
                                              DML.
 -T,--export-type <export-type>               The target type(none, jdbc,
                                              csv, hive) of export,
                                              default is jdbc.
 -t,--table <table-name>                      Table to populate.
 -u,--username <username>                     Set authentication username.
    --update-sql <update-sql>                 The self-define SQL for
                                              update operation.
 -v,--verbose <verbose>                       Print more information while
                                              working.
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
