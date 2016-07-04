---
title: Mac 使用Beyond Compare的几种Diff方式
toc: true
mathjax: true
date: 2016-06-08 17:58:07
categories: Mac常用技巧
tags: Mac常用技巧
description: 
---
作为一个Coder不管是QA还是RD在很多时候都需要使用diff工具。目前来说业界做的最好的工具也就是 ``` Beyond Compare``` 了，并且支持Windows和Mac,本文将介绍Mac系统下 ``` Beyond Compare``` 和 <font color=red>```Intellij idea``` ```PyCharm``` ```SourceTree``` ```Git客户端```</font> 等工具结合，diff代码的方法。
<!--more-->
# Beyond Compare的安装配置
**1 下载安装**
Beyond Compare 的官方下载地址为：[http://www.scootersoftware.com/download.php](http://www.scootersoftware.com/download.php)
下载下来的是dmg文件，然后双击安装一路next到底。
**2 快捷配置**
安装完成之后可以在Docker工具栏双击打开，但是很多使用Mac的同学，都喜欢使用命令行，所以可以建立一个命令行打开的快捷方式。方法如下：
打开Terminal终端输入如下的命令：  
```shell
ln -s /Applications/Beyond\ Compare.app/Contents/MacOS/bcomp /usr/local/bin/
``` 
以后打开 ``` Beyond Compare ``` 就可以直接在命令行输入: ``` bcomp ``` 打开了。
# Beyond Compare和其他工具的结合
## Beyond Compare和Intellij的结合
**1 打开Intelij Idea的设置找到如下的配置界面：**
![](http://7xutce.com1.z0.glb.clouddn.com/mac%2Fbeyond%2Fbeyond_compare_intellij_idea1.png)
**2 填入内容如下：**
- 勾选所有的复选框。
- Path to executable 填入内容如下： ```/usr/local/bin/bcomp ```。
- Parameters会默认填上参数，如果没有填上，请按照图中所示填充。

**3 使用步骤** 
3.1 点击左下角的代码的分支，会出现分支的列表，然后选择你要比较的分支，会出现Compare选项，如下图所示：
![](http://7xutce.com1.z0.glb.clouddn.com/mac%2Fbeyond%2Fbeyond_compare_intellij_idea3.png)
3.2 点击 Compare 选项会出现如下图所示的对话框,选择“diff”选项
![](http://7xutce.com1.z0.glb.clouddn.com/mac%2Fbeyond%2Fbeyond_compare_intellij_idea4.png)
3.3 点击任意一个文件，最后都会调用Beyond Compare来进行diff效果如下：
![](http://7xutce.com1.z0.glb.clouddn.com/mac%2Fbeyond%2Fbeyond_compare_intellij_idea5.png)
## Beyond Compare和PyCharm的结合
``` PyCharm ``` 和 ``` Intellij Idea ``` 都是 ``` JetBrains ``` 公司出品的优秀软件，两者的风格完全一样。所以配置  ``` Beyond Diff ``` 的过程完全一样。
## Beyond Compare和Source Tree的结合
