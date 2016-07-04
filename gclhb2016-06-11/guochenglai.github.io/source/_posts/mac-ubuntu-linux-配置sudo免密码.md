---
title: mac/ubuntu/linux 配置sudo免密码
date: 2016-05-30 16:24:29
categories: Mac常用技巧
tags:
- 免密码
- Mac常用技巧
description:
---
Mac/Ubuntu/Linux 配置sudo免密码只需要如下两部：
1 打开命令窗口输入如下命令：
```shell
sudo visudo 或者  sudo vi /etc/sudoers
```
<!-- more -->
2 替换 ``` #%admin  ALL=(ALL) ALL ``` 为 </br>``` %admin  ALL=(ALL) NOPASSWD: NOPASSWD: ALL```

