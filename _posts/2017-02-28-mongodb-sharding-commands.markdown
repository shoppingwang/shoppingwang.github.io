---
layout:     post
title:      "MongoDB Sharding Commands"
subtitle:   "For details on specific commands, including syntax and examples."
date:       2017-02-28
author:     "xp"
header-img: "img/post-sample-image.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Resource
---
* This will become a table of contents (this text will be scraped).
{:toc}

## [addShard](https://docs.mongodb.com/manual/reference/command/addShard/)

### Definition

* Adding a single database instance as a shard

    > { addShard: "\<hostname\>\<:port\>", maxSize: \<size\>, name: "\<shard_name\>" }

* Adding a replica set as a shard

    > { addShard: "\<replica_set\>/\<hostname\>\<:port\>", maxSize: \<size\>, name: "\<shard_name\>" }

### Examples

* The following command adds the database instance running on port **27027** on the host **mongodb0.example.net** as a shard

    > use admin
    > 
    > db.runCommand({addShard: "mongodb0.example.net:27027"})
    
* The following command adds a replica set as a shard

    > use admin
    >
    > db.runCommand( { addShard: "repl0/mongodb3.example.net:27327"} )
    
## [enableSharding](https://docs.mongodb.com/manual/reference/command/enableSharding/)

The enableSharding command enables sharding on a per-database level.

> use admin
> 
> db.runCommand({ enableSharding: "\<database name\>" })

## [listShards](https://docs.mongodb.com/manual/reference/command/listShards/)

### Definition

* Returns a list of the configured shards

    > { listShards: 1 }
    
### Example

* The following example returns the list of shards

    > use admin
    > 
    > db.getSiblingDB("admin").runCommand( { listShards: 1 } )
    
## [shardCollection](https://docs.mongodb.com/manual/reference/command/shardCollection/)

### Definition

* To run shardCollection, use the db.runCommand( { <command> } ) method

    > {
    > 
    > shardCollection: "\<database\>.\<collection\>",
    >
    > key: \<shardkey\>,
    >
    > unique: \<boolean\>,
    >
    > numInitialChunks: \<integer\>,
    >
    > collation: { locale: "simple" }
    >
    > }

### Example

* The following operation enables sharding for the **people** collection in the records database and uses the **zipcode** field as the _shard key_

    > use admin
    >
    > db.runCommand( { shardCollection: "records.people", key: { zipcode: 1 } } )