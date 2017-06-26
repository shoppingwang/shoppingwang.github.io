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

