---
layout:     post
title:      "Greenplum System Catalogs Definitions"
subtitle:   ""
date:       2017-02-07
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

The **gp\_db\_interfaces** table contains information about the relationship of segments to network interfaces. This information, joined with data from *gp\_interfaces*, is used by the system to optimize the usage of available network interfaces for various purposes, including fault detection.