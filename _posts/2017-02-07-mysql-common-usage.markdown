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

## 查看MYISAM引擎索引存储占用空间
```sql
-- 配置key_buffer_size参数大小
SELECT CONCAT(ROUND(SUM(INDEX_LENGTH) / 1024 / 1024 / 1024, 2), 'G') FROM information_schema.TABLES
WHERE ENGINE = 'MYISAM';
```

## 查看全局参数信息
```sql
SHOW GLOBAL VARIABLES LIKE 'myisam_sort_buffer_size';
```

## 查看各库中表的数据量和数据大小信息
```sql
SELECT TABLE_SCHEMA, TABLE_NAME, SUM(TABLE_ROWS) AS all_rows, SUM(DATA_LENGTH + INDEX_LENGTH) AS all_length
FROM INFORMATION_SCHEMA.TABLES
GROUP BY TABLE_SCHEMA, TABLE_NAME
ORDER BY all_rows DESC, all_length DESC;
```

## MySQL分区信息查看
```sql
SELECT partition_name, partition_expression, partition_description, table_rows
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = schema() AND TABLE_NAME='table';
```

## 查看正在使用中的表
```sql
SHOW OPEN TABLES where IN_USE > 0;
```

## 查看处理列表信息
```sql
SELECT id, user, host, db, command, time, state, info
FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE command <> 'sleep'
ORDER BY time DESC;
```

## 查看创建的索引的CARDINALITY比率
```sql
SELECT
    T1.TABLE_SCHEMA,
    T1.TABLE_NAME,
    T2.INDEX_NAME,
    ROUND(T2.CARDINALITY / T1.TABLE_ROWS * 100, 2) AS RATE
FROM
    INFORMATION_SCHEMA.TABLES T1,
    INFORMATION_SCHEMA.STATISTICS T2
WHERE
    T1.TABLE_SCHEMA = T2.TABLE_SCHEMA
        AND T1.TABLE_NAME = T2.TABLE_NAME
        AND T2.SEQ_IN_INDEX = (SELECT
            MIN(T3.SEQ_IN_INDEX)
        FROM
            INFORMATION_SCHEMA.STATISTICS T3
        WHERE
                T2.TABLE_NAME = T3.TABLE_NAME
                AND T2.TABLE_SCHEMA = T3.TABLE_SCHEMA
                AND T2.INDEX_NAME = T3.INDEX_NAME)
AND T1.TABLE_SCHEMA NOT IN ('MYSQL','PERFORMANCE_SCHEMA','INFORMATION_SCHEMA','SYS')
AND T1.TABLE_ROWS >=100
ORDER BY RATE;
```

## 查看锁阻塞
```sql
SELECT
    t3.trx_id waiting_trx_id,
    t3.trx_mysql_thread_id waiting_thread,
    t3.trx_query waiting_query,
    t2.trx_id blocking_trx_id,
    t2.trx_mysql_thread_id blocking_thread,
    t2.trx_query blocking_query
FROM
    information_schema.innodb_lock_waits t1,
    information_schema.innodb_trx t2,
    information_schema.innodb_trx t3
WHERE
    t1.blocking_trx_id = t2.trx_id
    AND t1.requesting_trx_id = t3.trx_id;
```

## 查询出哪些表不是InnoDB引擎的
```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_TYPE,
    ENGINE,
    CREATE_TIME,
    UPDATE_TIME,
    TABLE_COLLATION
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    TABLE_SCHEMA NOT IN ('information_schema' , 'mysql', 'performance_schema', 'sys')
    AND ENGINE <> 'InnoDB';
```

## 生成修改存储引擎的语句
```sql
SELECT
    -- TABLE_SCHEMA,
    -- TABLE_NAME,
    -- TABLE_TYPE,
    -- ENGINE,
    -- CREATE_TIME,
    -- UPDATE_TIME,
    -- TABLE_COLLATION,
     CONCAT('alter table ', TABLE_SCHEMA,'.',TABLE_NAME, ' engine=InnoDB;') AS alter_sql
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql', 'performance_schema', 'sys')
  AND ENGINE <> 'InnoDB';
```

## 查看会话连接信息
```sql
SELECT
    THREAD_ID,
    name,
    type,
    PROCESSLIST_ID,
    PROCESSLIST_USER AS user,
    PROCESSLIST_HOST AS host,
    PROCESSLIST_DB AS db,
    PROCESSLIST_COMMAND AS cmd,
    PROCESSLIST_TIME AS time,
    PROCESSLIST_STATE AS state,
    PROCESSLIST_INFO AS info,
    CONNECTION_TYPE AS type,
    THREAD_OS_ID AS os_id
FROM
    performance_schema.threads
WHERE
    type = 'FOREGROUND'
ORDER BY THREAD_ID;
```

## 查看数据库支持的字符集
```sql
SELECT * FROM INFORMATION_SCHEMA.CHARACTER_SETS
WHERE CHARACTER_SET_NAME LIKE 'utf%';

SHOW CHARACTER SET LIKE 'utf%';
```

## 查看数据库支持的排序字符集
```sql
-- 用于指定数据集如何排序，以及字符串的比对规则
SELECT * FROM INFORMATION_SCHEMA.COLLATIONS
WHERE COLLATION_NAME LIKE 'utf%';

SHOW COLLATION LIKE 'utf%';
```

## 查看表结构定义信息
```sql
SELECT
    table_name,
    COLUMN_NAME,
    ordinal_position,
    DATA_TYPE,
    IS_NULLABLE,
    COLUMN_DEFAULT,
    column_type,
    column_key,
    character_set_name,
    collation_name
FROM
    INFORMATION_SCHEMA.COLUMNS
WHERE
    table_name = 'employees'
        AND table_schema = 'employees';
SHOW COLUMNS FROM employees FROM employees;

DESC employeees.employees;
```

## 查看支持的引擎
```sql
SELECT *  FROM INFORMATION_SCHEMA.ENGINES;

SHOW ENGINES;
```

## 查看数据库的数据文件信息
```sql
SELECT
    FILE_ID,
    FILE_NAME,
    FILE_TYPE,
    TABLESPACE_NAME,
    FREE_EXTENTS,
    TOTAL_EXTENTS,
    ((TOTAL_EXTENTS - FREE_EXTENTS) * EXTENT_SIZE) / 1024 / 1024 AS MB_used,
    EXTENT_SIZE,
    INITIAL_SIZE,
    MAXIMUM_SIZE,
    AUTOEXTEND_SIZE,
    DATA_FREE,
    STATUS,
    ENGINE
FROM INFORMATION_SCHEMA.FILES;
```

## 查看指定表的约束
```sql
SELECT
    constraint_schema,
    table_name,
    constraint_name,
    column_name,
    ordinal_position,
    CONCAT(table_name,
            '.',
            column_name,
            ' -> ',
            referenced_table_name,
            '.',
            referenced_column_name) AS list_of_fks
FROM
    information_schema.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_SCHEMA = 'employees' AND REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY TABLE_NAME , COLUMN_NAME;
```

## 查看指定分区表信息
```sql
SELECT
    TABLE_SCHEMA,
    table_name,
    partition_name,
    subpartition_name sub_par,
    partition_ordinal_position par_position,
    partition_method method,
    partition_expression expression,
    partition_description description,
    table_rows
FROM information_schema.PARTITIONS
WHERE table_schema = 'test' AND table_name = 't';
```

## 查看支持的插件
```sql
SELECT
  PLUGIN_NAME, PLUGIN_STATUS, PLUGIN_TYPE,
  PLUGIN_LIBRARY, PLUGIN_LICENSE
FROM INFORMATION_SCHEMA.PLUGINS;

SHOW PLUGINS;
```

## 查看数据库中的存储过程、函数等
```sql
SELECT
    ROUTINE_SCHEMA,
    routine_name,
    ROUTINE_TYPE,
    data_type,
    routine_body,
    routine_definition,
    routine_comment
FROM
    INFORMATION_SCHEMA.ROUTINES
WHERE
    ROUTINE_TYPE = 'PROCEDURE'
AND ROUTINE_SCHEMA="employees";
```

## 查看存在的数据库及字符集信息
```sql
SELECT
    SCHEMA_NAME,
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM
    INFORMATION_SCHEMA.SCHEMATA;

SHOW DATABASES;
```

## 查看索引信息
```sql
SELECT
    table_schema,
    table_name,
    index_name,
    COLUMN_NAME,
    COLLATION,
    CARDINALITY,
    index_type
FROM
    INFORMATION_SCHEMA.STATISTICS
WHERE
    table_name = 'employees'
    AND table_schema = 'employees';

SHOW INDEX FROM employees FROM employees;
```

## 性能指标计算方式
```sql
-- 统计性能指标前先开启下列参数,该参数使用IS数据库来存放数据库信息,由于使用PS库存放还存在BUG,信息统计不全
show variables like 'show_compatibility_56';
set global show_compatibility_56=on;
show variables like 'show_compatibility_56';

-- QPS 计算(针对MyISAM引擎为主)
select variable_value into @v_questions from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Questions';
select variable_value into @v_uptime from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Uptime';
select round(@v_questions/@v_uptime,3) as "MyISAM/QPS";

-- QPS 计算(针对InnoDB引擎为主)
select variable_value into @v_com_update from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Com_update';
select variable_value into @v_com_select from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Com_select';
select variable_value into @v_com_insert from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Com_insert';
select variable_value into @v_com_delete from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Com_delete';
select variable_value into @v_uptime from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Uptime';
select round((@v_com_update+@v_com_select+@v_com_insert+@v_com_delete)/@v_uptime,3) as "InnoDB/QPS";

-- TPS 计算 (每秒事务数)
select variable_value into @v_com_commit from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Com_commit';
select variable_value into @v_com_rollback from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Com_rollback';
select variable_value into @v_uptime from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Uptime';
select round((@v_com_commit+@v_com_rollback)/@v_uptime,3) as "InnoDB/TPS (每秒事务数)";

-- InnoDB 缓存命中率
select variable_value into @v_read_requests from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Innodb_buffer_pool_read_requests';
select variable_value into @v_read_ahead from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Innodb_buffer_pool_read_ahead';
select variable_value into @v_reads from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Innodb_buffer_pool_reads';
select concat(round(@v_read_requests/(@v_read_requests+@v_read_ahead+@v_reads)*100,3),"%") as "InnoDB 缓存命中率";

-- Thread Cache命中率
select variable_value into @v_threads_created from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Threads_created';
select variable_value into @v_connections from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Connections';
select concat(round((1-@v_threads_created/@v_connections)*100,3),"%") as "线程缓存命中率";

-- 临时表使用情况
select variable_value into @v_Created_tmp_disk_tables from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Created_tmp_disk_tables';
select variable_value into @v_Created_tmp_files from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Created_tmp_files';
select variable_value into @v_Created_tmp_tables from INFORMATION_SCHEMA.global_status t1 where t1.variable_name='Created_tmp_tables';
select variable_value/1024/1024 into @v_tmp_table_size from INFORMATION_SCHEMA.GLOBAL_VARIABLES where VARIABLE_NAME='tmp_table_size';
select @v_tmp_table_size as "tmp_table_size(M)",@v_Created_tmp_disk_tables as Created_tmp_disk_tables,@v_Created_tmp_tables as Created_tmp_tables,@v_Created_tmp_files as Created_tmp_files,concat(round(@v_Created_tmp_disk_tables/@v_Created_tmp_tables*100,3),"%") as "临时表磁盘使用率";

-- 连接比率
select VARIABLE_VALUE into @v_max_conn from  INFORMATION_SCHEMA.GLOBAL_VARIABLES where VARIABLE_NAME='max_connections';
select VARIABLE_VALUE into @v_top_con from  INFORMATION_SCHEMA.GLOBAL_STATUS where VARIABLE_NAME='Max_used_connections';
select count(*) into @v_current_con from performance_schema.threads where type = 'FOREGROUND';
select @v_current_con as "当前连接数",@v_max_conn as "最大连接数",@v_top_con as "连接数最大峰值";

-- Innodb log buffer size的大小设置
select VARIABLE_VALUE/1024/1024 into @v_innodb_log_buffer_size from  INFORMATION_SCHEMA.GLOBAL_VARIABLES where VARIABLE_NAME='innodb_log_buffer_size';
select VARIABLE_VALUE into @Innodb_log_waits from  INFORMATION_SCHEMA.GLOBAL_STATUS where VARIABLE_NAME='Innodb_log_waits';
select @v_innodb_log_buffer_size as "日志缓存区大小(M)",@Innodb_log_waits as Innodb_log_waits;

-- 统计存储引擎分布情况
SELECT COUNT(*), engine FROM information_schema.TABLES WHERE table_schema NOT IN ('information_schema' , 'performance_schema', 'sys', 'mysql') GROUP BY engine;
SELECT COUNT(*),table_schema,engine FROM information_schema.TABLES WHERE table_schema NOT IN ('information_schema' , 'performance_schema', 'sys', 'mysql') GROUP BY table_schema,engine;
```