---
title: 修改mac下文件的默认打开软件
toc: true
date: 2016-05-30 14:52:22
categories: Mac常用技巧
tags: 
- Mac
- Mac常用技巧
description: 在我们打开mac文件的时候，系统会提供一个默认的软件来打开这个文件。但是在很多时候mac系统对文件提供的默认打开软件并不是我们想要的。例如：我有一个index.jade文件，双击该文件的时候我想用sublime打开但是打开的软件确实Xcode，导致我每次都要选择jade文件，右击在open with中选择自己想要的软件。这样确实太麻烦。本文将提供三种种方案来解决这个问题 ... 
---
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在我们打开mac文件的时候，系统会提供一个默认的软件来打开这个文件。但是在很多时候mac系统对文件提供的默认打开软件并不是我们想要的。例如：我有一个index.jade文件，双击该文件的时候我想用sublime打开但是打开的软件确实Xcode，导致我每次都要选择jade文件，右击在open with中选择自己想要的软件。这样确实太麻烦。本文将提供三种方案来解决这个问题。用户选择任意一个方案即可。
## 第一种方案--小白教程
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用操作界面修改文件的默认打开软件只需要如下三步：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1 找到想要打开的文件，右击选择 "get info"，出现如下图所示的对话框。
![](http://7xutce.com1.z0.glb.clouddn.com/mac%2Fmac_update_default_file_software.png?imageView2/2/w/200)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2 找到open with 如图红色方框，浏览找到目标软件。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3 关闭对话框，文件就会默认以你选择的软件打开。
## 第二种方案--命令行重度用户教程
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其实mac系统提供了一个命令行工具来打开各种软件。例如：
```shell
   open .  # 打开当前文件夹
   open test.txt # 以默认软件打开test.txt文件
   open -a MacDown test.md #以MacDown软件打开test.md文件
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;open命令可以帮助以任何的形式打开一个指定软件。但是每次需要使用该软件的软件名，使用其他略麻烦。下面提供了一个比较好的方案，执行如下命令：
```shell
ln -s /Applications/Sublime\ Text\ 2.app/Contents/SharedSupport/bin/subl /usr/local/bin/sublime
ln -s /Applications/MacDown.app/Contents/SharedSupport/bin/macdown /usr/local/bin/macdown
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;那么以后如果想要用sublime打开一个文件，可以使用如下命令：
```shell
sublime test.txt
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果想要用MacDown打开一个文件，可以使用如下的命令：
```shell
macdown test.md
```
## 第三种方案--爱钻研用户教程
郑重介绍一款软件duti:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;duti官方文档：[http://duti.org/documentation.html](http://duti.org/documentation.html)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;duti的github下载地址：[https://github.com/moretension/duti/releases/tag/duti-1.5.3](https://github.com/moretension/duti/releases/tag/duti-1.5.3)
duti提供了一种修改文件默认打开软件的新方法，首先给读者一个直观的感受：
**使用duti让所有对txt文件的操作都默认使用sublime**
```shell
 duti -s com.sublimetext.2 public.plain-text all
```
**上面的命令行可以这样理解：**
```shell
duti -s [软件的唯一标识] [文件的唯一标识] [软件对文件的操作权限]
```
那么软件唯一标识，文件唯一标识，软件对文件的操作权限又如何理解呢，请看下文：
1 软件唯一标识（Boundle ID）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;软件唯一标识是Mac系统为，每个软件提供的唯一标识，可以使用如下的命令获取一个软件的唯一标识：
```shell
MacBook-Pro-3:_posts guochenglai$ osascript -e 'id of app "Sublime Text 2"'
com.sublimetext.2
```
2 文件的唯一标识（UTI）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;UTI(Uniform type identifier)是mac系统提供的唯一标识，可以使用如下的命令获取一个文件的唯一标识：
```shell
MacBook-Pro-3:source guochenglai$ mdls 404.html
_kMDItemOwnerUserID            = 501
kMDItemContentCreationDate     = 2016-05-25 12:53:53 +0000
kMDItemContentModificationDate = 2016-05-25 12:53:53 +0000
kMDItemContentType             = "public.html"
kMDItemContentTypeTree         = (
    "public.html",
    "public.text",
    "public.data",
    "public.item",
    "public.content"
)
kMDItemDateAdded               = 2016-05-25 12:53:53 +0000
kMDItemDisplayName             = "404.html"
kMDItemFSContentChangeDate     = 2016-05-25 12:53:53 +0000
kMDItemFSCreationDate          = 2016-05-25 12:53:53 +0000
kMDItemFSCreatorCode           = ""
kMDItemFSFinderFlags           = 0
kMDItemFSHasCustomIcon         = (null)
kMDItemFSInvisible             = 0
kMDItemFSIsExtensionHidden     = 0
kMDItemFSIsStationery          = (null)
kMDItemFSLabel                 = 0
kMDItemFSName                  = "404.html"
kMDItemFSNodeCount             = (null)
kMDItemFSOwnerGroupID          = 20
kMDItemFSOwnerUserID           = 501
kMDItemFSSize                  = 366
kMDItemFSTypeCode              = ""
kMDItemKind                    = "HTML document"
kMDItemLogicalSize             = 366
kMDItemPhysicalSize            = 4096
```
可以看到 html文件的唯一标识是"public.html"</br>
3 软件对文件的造作权限
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;软件对文件的操作权限如下：
```shell 
all #该软件拥有该文件的所有操作权限
viewer #该软件拥有该文件的读权限
editor #该软件拥有该文件的写权限
shell #该软件拥有该文件的执行权限
none #该软件对该文件没有权限
```
通过三步的分析 ```duti -s com.sublimetext.2 public.plain-text all```就不用再解释了吧！！！