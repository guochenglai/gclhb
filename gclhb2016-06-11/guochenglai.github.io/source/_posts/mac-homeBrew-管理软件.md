---
title: mac homeBrew 管理软件
toc: true
date: 2016-05-30 18:20:22
categories: Mac常用技巧
tags:
- Mac常用技巧
- Mac环境配置
description: 在使用Ubuntu的时候经常使用<font color=red>sudo apt-get</font>命令来进行软件的安装、更新、删除等操作。那么在mac系统有没有一种类似的解决方案呢？在mac上面有一款类似的包管理工具他的名字叫：HomeBrew。
---
## homeBrew介绍
  Homebrew是Mac系统上强大的包管理器，为软件的安装提供了类似ubuntu系统上的<font color=red>apt-get</font>工具，他具有以下优点：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1 可以自动解决参数依赖问题
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2 安装软件不需要加sudo
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3 提供了一键式安装
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;homebrew官方地址：[http://brew.sh](http://brew.sh)
## homeBrew安装
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;安装home brew 只需要打开shell终端，输入以下命令：
```ruby
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"  
```
## homeBrew使用介绍
<font color=red>**本文所有的软件包以git为例。**</font>
1 bew安装git
```shell
brew install git 
```
2 bew查找git软件包
```shell
brew search git
```
3 brew列出所有安装的软件包
```shell
brew list
```
4 brew删除git软件包
```shell
brew remove git
```
5 brew查看软件包信息
```shell
brew info git
```
6 brew列出软件包的依赖关系
```shell
brew deps git
```
7 更新brew
```shell
brew update
```
......