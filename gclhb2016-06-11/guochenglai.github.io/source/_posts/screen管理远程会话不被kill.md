---
title: screen管理远程会话不被kill
toc: true
mathjax: true
date: 2016-06-07 20:06:00
categories: Linux常用技巧
tags: 
- Linux
- Linux常用技巧
description: 在我们的开发过程中经常需要 SSH 或者 telent 远程登录到 Linux 服务器。然后执行一些运行时间很长的任务。通常情况下我们都是为每一个这样的任务开一个远程终端窗口，因为他们执行的时间太长了。必须等待它执行完毕，在此期间可不能关掉窗口或者断开连接，否则这个任务就会被杀掉，一切半途而废了。最近终于找到了解决这个问题的神器-screen。
---
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在我们用shell终端，执行一个命令的时候，我们所执行的任务都是这个shell终端的子进程。如果因为用户注销或者断网，shell终端会收到系统发来的HUP信号，当shell终端收到HUP信号之后，会关闭shell终端的子进程以及，shell终端本身性能。所以解决大任务不被kill的方法有两种：1 让shell进行忽略HUP信号 2让进程不属于shell终端子进程。下面将介绍着几种方法。
## nohub
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;nohub的用途是让提交的命令忽略HUP信号，使用的时候只需要在我们的shell命令前面加入<font color=red>nohup</font>命令即可如下：
```
 nohup brew install mysql &
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;shell终端回显如下：
```
appending output to nohup.out
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;说明：这是一个安装MySQL的长任务，我们在shell命令前面加：nohub命令。然后执行任务的进程不再接受hup信号，即使shell终端被关闭，这个任务也可以继续执行。日志文件会输出到当前目录的nohup.out文件。
## 线程后台运行
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在程序后台运行的时候有一个常见的理解误区。例如：
```
ping www.baidu.com &
```


## screen
