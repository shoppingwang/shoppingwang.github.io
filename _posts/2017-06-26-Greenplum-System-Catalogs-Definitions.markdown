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

## pg_compression
描述系统可用压缩方法的系统目录表。

## pg_constraint
此系统表存储了表的检查、主键、唯一键和外键的约束信息。

The **pg_constraint** system catalog table stores check, primary key, unique, and foreign key constraints on tables. Column constraints are not treated specially. Every column constraint is equivalent to some table constraint. Not-null constraints are represented in the ***pg_attribute*** catalog table. Check constraints on domains are stored here, too.

## pg_conversion
此系统表存储了可用编码转换的存储过程。

The **pg_conversion** system catalog table describes the available encoding conversion procedures as defined by CREATE CONVERSION.

## pg_database
可用的数据库相关存储信息。

The **pg_database** system catalog table stores information about the available databases. Databases are created with the CREATE DATABASE SQL command. Unlike most system catalogs, pg_database is shared across all databases in the system. There is only one copy of pg_database per system, not one per database.

## pg_depend
此系统表记录了数据库之间的依赖关系。

The **pg_depend** system catalog table records the dependency relationships between database objects. This information allows `DROP` commands to find which other objects must be dropped by `DROP CASCADE` or prevent dropping in the `DROP RESTRICT` case. See also ***pg_shdepend***, which performs a similar function for dependencies involving objects that are shared across a Greenplum system.

## pg_description
此系统表对对每个数据库对象存储了可选的描述信息。

The **pg_description** system catalog table stores optional descriptions (comments) for each database object. Descriptions can be manipulated with the `COMMENT` command and viewed with `psql's` `\d` meta-commands. Descriptions of many built-in system objects are provided in the initial contents of **pg_description**. See also ***pg_shdescription***, which performs a similar function for descriptions involving objects that are shared across a Greenplum system.

## pg_exttable
此系统表用于追踪由`CREATE EXTERNAL TABLE`命令创建的外部表和WEB表。

## pg_filespace
此来不及包含了在GP系统中创建的文件空间信息。

The **pg_filespace** table contains information about the filespaces created in a Greenplum Database system. Every system contains a default filespace, **pg_system**, which is a collection of all the data directory locations created at system initialization time.

## pg\_filespace\_entry
文件空间需要一个系统位置去存储它的数据库文件。

A tablespace requires a file system location to store its database files. In Greenplum Database, the master and each segment (primary and mirror) needs its own distinct storage location. This collection of file system locations for all components in a Greenplum system is referred to as a filespace. The **pg\_filespace\_entry** table contains information about the collection of file system locations across a Greenplum Database system that comprise a Greenplum Database filespace.

## pg_index
此系统表包含了部分的索引信息，剩余的其他大部分在***pg_class***表中。

The **pg_index** system catalog table contains part of the information about indexes. The rest is mostly in ***pg_class***.

## pg_inherits
此系统表记录了表之间的继承的层次结构。

The **pg_inherits** system catalog table records information about table inheritance hierarchies. There is one entry for each direct child table in the database. (Indirect inheritance can be determined by following chains of entries.) In Greenplum Database, inheritance relationships are created by both the `INHERITS` clause (standalone inheritance) and the `PARTITION BY` clause (partitioned child table inheritance) of `CREATE TABLE`.

## pg_language
此系统表注册了你能在写函数或存储过程中所能使用的语言信息。

The **pg_language** system catalog table registers languages in which you can write functions or stored procedures. It is populated by `CREATE LANGUAGE`.

## pg_largeobject
此系统表持有了标记为大对象的数据。

The **pg_largeobject** system catalog table holds the data making up 'large objects'. A large object is identified by an OID assigned when it is created. Each large object is broken into segments or 'pages' small enough to be conveniently stored as rows in ***pg_largeobject***. The amount of data per page is defined to be `LOBLKSIZE` (which is currently `BLCKSZ`/4, or typically 8K).

## pg_listener
此系统表支持`LISTEN`和`NOTIFY`命令信息。

The **pg_listener** system catalog table supports the `LISTEN` and `NOTIFY` commands. A listener creates an entry in **pg_listener** for each notification name it is listening for. A notifier scans and updates each matching entry to show that a notification has occurred. The notifier also sends a signal (using the PID recorded in the table) to awaken the listener from sleep.

## pg_locks
此视图用来获取由GP数据库打开的事务锁的信息。

The **pg_locks** view provides access to information about the locks held by open transactions within Greenplum Database.

## pg_namespace
此系统表存储了命名空间信息。

The **pg_namespace** system catalog table stores namespaces. A namespace is the structure underlying SQL schemas: each namespace can have a separate collection of relations, types, etc. without name conflicts.

## pg_opclass
此系统表定义了操作类获取方法的目录信息。

The **pg_opclass** system catalog table defines index access method operator classes. Each operator class defines semantics for index columns of a particular data type and a particular index access method. Note that there can be multiple operator classes for a given data type/access method combination, thus supporting multiple behaviors. The majority of the information defining an operator class is actually not in its **pg_opclass** row, but in the associated rows in ***pg_amop*** and ***pg_amproc***. Those rows are considered to be part of the operator class definition — this is not unlike the way that a relation is defined by a single ***pg_class*** row plus associated rows in ***pg_attribute*** and other tables.

## pg_operator
此系统表存储了操作符的相关信息，包含内建的和通过`CREATE OPERATOR`命令创建的操作符。

The **pg_operator** system catalog table stores information about operators, both built-in and those defined by `CREATE OPERATOR`. Unused column contain zeroes. For example, oprleft is zero for a prefix operator.

## pg_partition
此系统表用来追踪分区表以及它们之间的继承关系。

The **pg_partition** system catalog table is used to track partitioned tables and their inheritance level relationships. Each row of **pg_partition** represents either the level of a partitioned table in the partition hierarchy, or a subpartition template description. The value of the attribute paristemplate determines what a particular row represents.

## pg_partition_columns
此系统视图用来展示分区表的分区列的相关信息。

## pg_partition_encoding
此系统表描述了对分区模板所能使用的压缩选项信息。

## pg_partition_rule
此系统目录表用来追踪分区表它们的检查约束和数据规则信息。

The **pg_partition_rule** system catalog table is used to track partitioned tables, their check constraints, and data containment rules. Each row of **pg_partition_rule** represents either a leaf partition (the bottom level partitions that contain data), or a branch partition (a top or mid-level partition that is used to define the partition hierarchy, but does not contain any data).

## pg\_partition\_templates
此系统视图用来展示由子分区模板创建的子分区信息。

## pg_partitions
此系统视图用来展示分区表的结构信息。

## pg_pltemplate
此系统目录表存储了存储过程的模板信息。

The **pg_pltemplate** system catalog table stores template information for procedural languages. A template for a language allows the language to be created in a particular database by a simple `CREATE LANGUAGE` command, with no need to specify implementation details. Unlike most system catalogs, **pg_pltemplate** is shared across all databases of Greenplum system: there is only one copy of **pg_pltemplate** per system, not one per database. This allows the information to be accessible in each database as it is needed.

## pg_proc
此系统目录表存储了函数的相关信息，包含内建的函数和使用`CREATE FUNCTION`创建的函数。

The **pg_proc** system catalog table stores information about functions (or procedures), both built-in functions and those defined by `CREATE FUNCTION`. The table contains data for aggregate and window functions as well as plain functions. If proisagg is true, there should be a matching row in ***pg_aggregate***. If proiswin is true, there should be a matching row in ***pg_window***.

## pg_resourcetype
此系统目录表包含了GP资源队列的扩展属性信息。

The **pg_resourcetype** system catalog table contains information about the extended attributes that can be assigned to Greenplum Database resource queues. Each row details an attribute and inherent qualities such as its default setting, whether it is required, and the value to disable it (when allowed).

## pg_resqueue
此系统目录表包含了GP数据库资源队列负载管理参数的相关信息。

The **pg_resqueue** system catalog table contains information about Greenplum Database resource queues, which are used for the workload management feature. This table is populated only on the master. This table is defined in the ***pg_global*** tablespace, meaning it is globally shared across all databases in the system.

## pg\_resqueue\_attributes
此视图允许管理员查看资源队列，如它的同时活动语句数限制、查询资源限制、优先级等。

## pg_resqueuecapability
此系统目录表包含GP数据库资源队列的扩展属性、或者容量信息。

The **pg_resqueuecapability** system catalog table contains information about the extended attributes, or capabilities, of existing Greenplum Database resource queues. Only resource queues that have been assigned an extended capability, such as a priority setting, are recorded in this table. This table is joined to the ***pg_resqueue*** table by resource queue object ID, and to the ***pg_resourcetype*** table by resource type ID (`restypid`).

## pg_rewrite
此系统目录表存储了表和视图的重写规则。

The **pg_rewrite** system catalog table stores rewrite rules for tables and `views.pg_class.relhasrules` must be true if a table has any rules in this catalog.

## pg_roles
此视图提供数据库角色的访问信息。

The view **pg_roles** provides access to information about database roles. This is simply a publicly readable view of ***pg_authid*** that blanks out the password field. This view explicitly exposes the OID column of the underlying table, since that is needed to do joins to other catalogs.

## pg_shdepend
此系统目录表记录了数据库对象和共享对象之间的依赖关系，像角色之类的。

The **pg_shdepend** system catalog table records the dependency relationships between database objects and shared objects, such as roles. This information allows Greenplum Database to ensure that those objects are unreferenced before attempting to delete them. See also ***pg_depend***, which performs a similar function for dependencies involving objects within a single database. Unlike most system catalogs, **pg_shdepend** is shared across all databases of Greenplum system: there is only one copy of **pg_shdepend** per system, not one per database.

## pg_shdescription
此系统表存储了共享数据库对象的可选描述（备注）信息。

The **pg_shdescription** system catalog table stores optional descriptions (comments) for shared database objects. Descriptions can be manipulated with the `COMMENT` command and viewed with `psql`'s `\d` meta-commands. See also ***pg_description***, which performs a similar function for descriptions involving objects within a single database. Unlike most system catalogs, **pg_shdescription** is shared across all databases of a Greenplum system: there is only one copy of **pg_shdescription** per system, not one per database.

## pg\_stat\_activity
此视图展示了每个服务进程、相关的用户会话详情、查询信息。

The view **pg\_stat\_activity** shows one row per server process and details about it associated user session and query. The columns that report data on the current query are available unless the parameter `stats\_command\_string` has been turned off. Furthermore, these columns are only visible if the user examining the view is a superuser or the same as the user owning the process being reported on.

## pg\_stat\_last\_operation
此表追踪数据库对象的元数据信息。

## pg\_stat\_last\_shoperation
此表追踪全局对象的元数据信息。

## pg\_stat\_operations
此视图展示了在数据库对象上执行的最后的操作详细信息。

## pg\_stat\_partition\_operations
此视图展示了在分区表上执行的最后的操作信息。

## pg\_stat\_replication
此视图包含用来GP数据库主节点映像操作的walsender进程的元数据信息。

## pg_statistic
此系统目录表存储了数据库内容的统计数据信息。

The **pg_statistic** system catalog table stores statistical data about the contents of the database. Entries are created by `ANALYZE` and subsequently used by the query optimizer. There is one entry for each table column that has been analyzed. Note that all the statistical data is inherently approximate, even assuming that it is up-to-date.

**pg_statistic** also stores statistical data about the values of index expressions. These are described as if they were actual data columns; in particular, starelid references the index. No entry is made for an ordinary non-expression index column, however, since it would be redundant with the entry for the underlying table column.

## pg\_stat\_resqueues
此视图允许管理员查通过时间查看资源队列的指标信息。

The **pg\_stat\_resqueues** view allows administrators to view metrics about a resource queue's workload over time. To allow statistics to be collected for this view, you must enable the `stats_queue_level` server configuration parameter on the Greenplum Database master instance. Enabling the collection of these metrics does incur a small performance penalty, as each statement submitted through a resource queue must be logged in the system catalog tables.

## pg\_stat\_resqueues
此视图允许用户通过时间查看资源队列的负载指标。

The **pg\_stat\_resqueues** view allows administrators to view metrics about a resource queue's workload over time. To allow statistics to be collected for this view, you must enable the `stats_queue_level` server configuration parameter on the Greenplum Database master instance. Enabling the collection of these metrics does incur a small performance penalty, as each statement submitted through a resource queue must be logged in the system catalog tables.

## pg_tablespace
此系统目录表存储有可用的表空间信息。

The **pg_tablespace** system catalog table stores information about the available tablespaces. Tables can be placed in particular tablespaces to aid administration of disk layout. Unlike most system catalogs, **pg_tablespace** is shared across all databases of a Greenplum system: there is only one copy of **pg_tablespace** per system, not one per database.

## pg_trigger
此系统目录表存储表上的触发器信息。
> GP不支持触发器。

## pg_type
此系统目录表存储数据类型相关的信息。

The **pg_type** system catalog table stores information about data types. Base types (scalar types) are created with `CREATE TYPE`, and domains with `CREATE DOMAIN`. A composite type is automatically created for each table in the database, to represent the row structure of the table. It is also possible to create composite types with `CREATE TYPE AS`.

## pg\_type\_encoding
此系统目录表包含列的存储类型信息。

## pg\_user\_mapping
此目录表存储本地用户到远程用户的映射。必须有管理员权限才能查看此目录表。

## pg_window
此表存储了窗口函数的相关信息。

The **pg_window** table stores information about window functions. Window functions are often used to compose complex OLAP (online analytical processing) queries. Window functions are applied to partitioned result sets within the scope of a single query expression. A window partition is a subset of rows returned by a query, as defined in a special `OVER()` clause. Typical window functions are `rank`, `dense_rank`, and `row_number`. Each entry in **pg_window** is an extension of an entry in ***pg_proc***. The ***pg_proc*** entry carries the window function's name, input and output data types, and other information that is similar to ordinary functions.
