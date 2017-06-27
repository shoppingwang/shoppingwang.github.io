---
layout:     post
title:      "Greenplum System Catalogs Definitions"
subtitle:   ""
date:       2017-06-26
author:     "xp"
header-img: "img/post-bg-alitrip.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Greenplum
    - Tool
---

## gp\_configuration\_history
包含错误检测和恢复操作时所做的系统变更。

The **gp\_configuration\_history** table contains information about system changes related to fault detection and recovery operations. The fts_probe process logs data to this table, as do certain related management utilities such as `gpcheck`, `gprecoverseg`, and `gpinitsystem`. For example, when you add a new segment and mirror segment to the system, records for these events are logged to gp\_configuration\_history.

## gp\_db\_interfaces
包含segmengs节点网络相关的一些信息。

The **gp\_db\_interfaces** table contains information about the relationship of segments to network interfaces. This information, joined with data from ***gp\_interfaces***, is used by the system to optimize the usage of available network interfaces for various purposes, including fault detection.

## gp\_distributed\_log
此视图包含分布式事务和本地事务的一些相关信息。

The **gp\_distributed\_log** view contains status information about distributed transactions and their associated local transactions. A distributed transaction is a transaction that involves modifying data on the segment instances. Greenplum's distributed transaction manager ensures that the segments stay in synch. This view allows you to see the status of distributed transactions.

## gp\_distributed\_xacts
此视图包含GP的当前活动的分布式事务信息。

The **gp\_distributed\_xacts** view contains information about Greenplum Database distributed transactions. A distributed transaction is a transaction that involves modifying data on the segment instances. Greenplum's distributed transaction manager ensures that the segments stay in synch. This view allows you to see the currently active sessions and their associated distributed transactions.

## gp\_distribution\_policy
包含GP表数据在segments节点上的数据分布策略信息。

The **gp\_distribution\_policy** table contains information about Greenplum Database tables and their policy for distributing table data across the segments. This table is populated only on the master. This table is not globally shared, meaning each database has its own copy of this table.

## gpexpand.expansion_progress
此视图包含了系统节点扩展的进度信息。

The **gpexpand.expansion_progress** view contains information about the status of a system expansion operation. The view provides calculations of the estimated rate of table redistribution and estimated time to completion.

Status for specific tables involved in the expansion is stored in ***gpexpand.status_detail***.

## gpexpand.status
包含系统节点扩展的状态信息。

The **gpexpand.status** table contains information about the status of a system expansion operation. Status for specific tables involved in the expansion is stored in gpexpand.status_detail.

In a normal expansion operation it is not necessary to modify the data stored in this table.

## gpexpand.status_detail
包含调用系统节点扩展操作时表的状态信息。

The **gpexpand.status_detail** table contains information about the status of tables involved in a system expansion operation. You can query this table to determine the status of tables being expanded, or to view the start and end time for completed tables.

## gp_fastsequence
包含可追加、面向列的表的最大的行号。

The **gp_fastsequence** table contains information about append-optimized and column-oriented tables. The last_sequence value indicates maximum row number currently used by the table.

## gp\_fault\_strategy
包含特定的错误活动。

The **gp\_fault\_strategy** table specifies the fault action.

## gp\_global\_sequence
包含事务日志中日志序列号位置，主要用于节点间的文件块复制。

The **gp\_global\_sequence** table contains the log sequence number position in the transaction log, which is used by the file replication process to determine the file blocks to replicate from a primary to a mirror segment.

## gp_id
包含GP系统节点的名称和序号。

The **gp_id** system catalog table identifies the Greenplum Database system name and number of segments for the system. It also has local values for the particular database instance (segment or master) on which the table resides. This table is defined in the pg_global tablespace, meaning it is globally shared across all databases in the system.

## gp_interfaces
包含segment主机节点的网络接口信息。

The **gp_interfaces** table contains information about network interfaces on segment hosts. This information, joined with data from ***gp\_db\_interfaces***, is used by the system to optimize the usage of available network interfaces for various purposes, including fault detection.

## gp\_persistent\_database\_node
包含追踪数据库对象相关的事务状态的系统文件对象的状态。

The **gp\_persistent\_database\_node** table keeps track of the status of file system objects in relation to the transaction status of database objects. This information is used to make sure the state of the system catalogs and the file system files persisted to disk are synchronized. This information is used by the primary to mirror file replication process.

## gp\_persistent\_relation\_node
包含追踪关联对象（表、视图、索引等等）相关的事务状态的系统文件对象的状态。

The **gp\_persistent\_relation\_node** table table keeps track of the status of file system objects in relation to the transaction status of relation objects (tables, view, indexes, and so on). This information is used to make sure the state of the system catalogs and the file system files persisted to disk are synchronized. This information is used by the primary to mirror file replication process.

## gp\_persistent\_tablespace\_node
包含追踪表空间对象相关的事务状态的系统文件对象的状态。

The **gp\_persistent\_tablespace\_node** table keeps track of the status of file system objects in relation to the transaction status of tablespace objects. This information is used to make sure the state of the system catalogs and the file system files persisted to disk are synchronized. This information is used by the primary to mirror file replication process.

## gp_pgdatabase
此视图包含GP节点实例的状态信息，无论此实例是镜像节点还是主节点。

The **gp_pgdatabase** view shows status information about the Greenplum segment instances and whether they are acting as the mirror or the primary. This view is used internally by the Greenplum fault detection and recovery utilities to determine failed segments.

## gp\_relation\_node
包含系统对象（表、视图、索引等等）之间的关系。

The **gp\_relation\_node** table contains information about the file system objects for a relation (table, view, index, and so on).

## gp\_resqueue\_status
此视图允许管理员查看资源队列的负载状态和活动信息。

The **gp_toolkit.gp\_resqueue\_status** view allows administrators to see status and activity for a workload management resource queue. It shows how many queries are waiting to run and how many queries are currently active in the system from a particular resource queue.

## gp\_san\_configuration
包含SAN的挂载点信息。

The **gp\_san\_configuration** table contains mount-point information for SAN failover.

## gp\_segment\_configuration
包含节点和镜像的配置信息。

The **gp\_segment\_configuration** table contains information about mirroring and segment configuration.

## gp\_transaction\_log
此视图包含部分节点的本地事务状态信息。

The **gp\_transaction\_log** view contains status information about transactions local to a particular segment. This view allows you to see the status of local transactions.

## gp\_version\_at\_initdb
记录了GP数据库第一次初始化时的版本信息。

The **gp\_version\_at\_initdb** table is populated on the master and each segment in the Greenplum Database system. It identifies the version of Greenplum Database used when the system was first initialized. This table is defined in the ***pg_global*** tablespace, meaning it is globally shared across all databases in the system.

## pg_aggregate
包含聚合函数信息。

The **pg_aggregate** table stores information about aggregate functions. An aggregate function is a function that operates on a set of values (typically one column from each row that matches a query condition) and returns a single value computed from all these values. Typical aggregate functions are `sum`, `count`, and `max`. Each entry in **pg_aggregate** is an extension of an entry in ***pg_proc***. The ***pg_proc*** entry carries the aggregate's name, input and output data types, and other information that is similar to ordinary functions.

## pg_am
包含索引的存储方式信息。

The **pg_am** table stores information about index access methods. There is one row for each index access method supported by the system.

## pg_amop
此表存储有关索引存储方法操作类的相关操作信息。

The **pg_amop** table stores information about operators associated with index access method operator classes. There is one row for each operator that is a member of an operator class.

## pg_amproc
此表存储支持索引存储方法操作类的存储过程信息。

The **pg_amproc** table stores information about support procedures associated with index access method operator classes. There is one row for each support procedure belonging to an operator class.

## pg_appendonly
包含可追加优化表的存储选项和其他的特征信息。

The **pg_appendonly** table contains information about the storage options and other characteristics of append-optimized tables.

## pg_attrdef
此表存储了列的默认值信息。

The **pg_attrdef** table stores column default values. The main information about columns is stored in ***pg_attribute***. Only columns that explicitly specify a default value (when the table is created or the column is added) will have an entry here.

## pg_attribute
此表存储表的列信息。

The **pg_attribute** table stores information about table columns. There will be exactly one pg_attribute row for every column in every table in the database. (There will also be attribute entries for indexes, and all objects that have pg_class entries.) The term attribute is equivalent to column.

## pg\_attribute\_encoding
此系统目录表包含列的存储信息。

The **pg\_attribute\_encoding** system catalog table contains column storage information.

## pg\_auth\_members
此系统目录表展示了系统角色之间的关系。

The **pg\_auth\_members** system catalog table shows the membership relations between roles. Any non-circular set of relationships is allowed. Because roles are system-wide, **pg\_auth\_members** is shared across all databases of a Greenplum Database system.

## pg_authid
包含了数据库授权标识信息。

The **pg_authid** table contains information about database authorization identifiers (roles). A role subsumes the concepts of users and groups. A user is a role with the rolcanlogin flag set. Any role (with or without rolcanlogin) may have other roles as members. See ***pg\_auth\_members***.

Since this catalog contains passwords, it must not be publicly readable. ***pg_roles*** is a publicly readable view on **pg_authid** that blanks out the password field.

Because user identities are system-wide, pg_authid is shared across all databases in a Greenplum Database system: there is only one copy of **pg_authid** per system, not one per database.

## pg_cast
此表存储了数据类型的转换路径信息，包含系统内建的和通过CREATE CAST命令进行定义的路径。

The **pg_cast** table stores data type conversion paths, both built-in paths and those defined with `CREATE CAST`. The cast functions listed in **pg_cast** must always take the cast source type as their first argument type, and return the cast destination type as their result type. A cast function can have up to three arguments. The second argument, if present, must be type integer; it receives the type modifier associated with the destination type, or -1 if there is none. The third argument, if present, must be type boolean; it receives `true` if the cast is an explicit cast, `false` otherwise.

It is legitimate to create a `pg_cast` entry in which the source and target types are the same, if the associated function takes more than one argument. Such entries represent 'length coercion functions' that coerce values of the type to be legal for a particular type modifier value. Note however that at present there is no support for associating non-default type modifiers with user-created data types, and so this facility is only of use for the small number of built-in types that have type modifier syntax built into the grammar.

When a **pg_cast** entry has different source and target types and a function that takes more than one argument, it represents converting from one type to another and applying a length coercion in a single step. When no such entry is available, coercion to a type that uses a type modifier involves two steps, one to convert between data types and a second to apply the modifier.

## pg_class
系统目录表pg_class对表和大部分其他具有列或与表类似的其他目录进行编目。

The system catalog table **pg_class** catalogs tables and most everything else that has columns or is otherwise similar to a table (also known as relations). This includes indexes (see also ***pg_index***), sequences, views, composite types, and `TOAST` tables. Not all columns are meaningful for all relation types.