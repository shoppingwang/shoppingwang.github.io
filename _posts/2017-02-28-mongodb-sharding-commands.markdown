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

### [addShard](https://docs.mongodb.com/manual/reference/command/addShard/)

* Adding a single database instance as a shard
    ```json
    { addShard: "<hostname><:port>", maxSize: <size>, name: "<shard_name>" }
    ```
* Adding a replica set as a shard