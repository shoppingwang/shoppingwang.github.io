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
    - Sublime
    - 文件处理
---

> 一般在Linux下会使用sed命令来批量处理文件，这里我们在Mac下使用Sublime进行批量文件替换。

配置截图如下：

![](/img/avc-oozie-send-mail-config.png)
- 使用`Shift+Command+F`命令调出替换窗口（`Find->Find in files...`）
- 在**Find**输入框中输入需要被替换的字符串（`Option+Command+R`可开启或关闭正则表达式模式匹配）
- 在**Where**输入框的最右选择你需要处理的文件夹
- 在**Replace**输入框中输入需要替换的字符串
- 执行**Replace**操作
- 使用`Option+Command+S`保存全部被更改的文件（`File->Save All`）
