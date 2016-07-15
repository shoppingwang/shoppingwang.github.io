---
layout:     post
title:      "如何在Oozie中配置Java邮件发送任务"
subtitle:   "使用Java调用Http请求发送邮件"
date:       2016-07-15
author:     "xp"
header-img: "img/home-bg-geek.jpg"
header-mask: 0.3
catalog:    true
tags:
    - AVCData
    - Hue
    - Oozie
    - Tool
---

> 在Oozie中配置Workflow时，针对任务处理的每一步，当任务失败后，失败的节点应流向邮件发送节点，即时发送邮件通知Workflow负责人进行处理。

### 配置截图

![](http://www.mllearn.com/img/avc-oozie-send-mail-config.png)

### 参数说明
1. 任务的名称：avc\_dis\_plat\_trend\_daily-send-email（Workflow名称-任务名称）
2. 执行的Jar路径：/user/avc_spark_etl/bin/etl-util-0.0.1-jar-with-dependencies.jar
3. 执行的主类：com.avcdata.etl.util.SendMail
4. 邮件发送地址：http://mail.example.com/send
5. 收件人地址，多个收件人用`,`分隔
6. 邮件主题，建议为`The user ${wf:user()}'s task ${wf:name()} failed`
7. 邮件内容，建议为`Task [User=${wf:user()}, TaskName=${wf:name()}] execute failed. Last error  message is [${wf:errorMessage(wf:lastErrorNode())}]. Execute time is [${timestamp()}].`
8. 依赖的HDFS的执行Jar路径：/user/avc_spark_etl/bin/etl-util-0.0.1-jar-with-dependencies.jar