---
title: MySQL索引优化分析，SQL优化，慢查询分析
toc: true
date: 2016-05-29 10:54:20
categories: Mysql 
tags:
- Mysql 
- 索引优化
- 慢查询分析
- 索引分析
description: 本文基于Mysql5.6主要介绍了MySQL的索引分析工具--explain和profiling。并利用MySQL的索引分析工具，对MySQL的索引进行分析，通过观察MySQL索引分析的过程，可以看到常见的索引优化点，以及在使用索引的时候的常见问题。本文的最后部分介绍了MySQL的无效索引（什么时候你为这个字段添加了索引，但是却无法使用）....
---
## *配置环境说明*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Mysql的版本信息：
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_db_version.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;操作系统版本信息：
  ![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explain_system_version.png?imageView2/2/w/400)
## *索引的分析*
### *数据的准备*
#### *数据库的建表SQL*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表的说明：id是自增主键，name是唯一索引，age 是非唯一索引，desc无索引。
```sql
CREATE TABLE `index_test` (  
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增ID',  
  `name` varchar(128) COLLATE utf8_bin NOT NULL DEFAULT '' COMMENT '名字',  
  `age` int(11) NOT NULL COMMENT '年龄',  
  `desc` varchar(128) CHARACTER SET utf8 NOT NULL DEFAULT '' COMMENT '描述',  
  `status` tinyint(4) NOT NULL COMMENT '状态',  
  PRIMARY KEY (`id`),  
  UNIQUE KEY `uniq_name` (`name`),  
  KEY `idx_age` (`age`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ; 
```
#### *表中测试数据*
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_test_data.png)
### *索引的分析*
#### *使用explain查看sql的执行计划*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在MySQL中可以在sql前面加上explain语句，来显示该条SQL的执行计划，输出内容如下：![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian.png)
#### *explain详解*
##### *select_type*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;select_type表示查询语句的类型，取值主要有以下几种:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;simple：表示是简单的单表查询
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;primary：表示子查询的外表
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;derived：派生表的查询
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_select_type_derived.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;subquery: 子查询的内部第一个SQL
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_select_type_subquery.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;union：表示union操作被连接的表  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;union result：表示连接操作之后的结果表  
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_select_type_union.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;depend union 表示子查询中union语句
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;depend subquery 表示子查询中生成的结果
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_select_type_dependent_subquery_union_union_all.png) 
##### *table*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当前SQL查询涉及到的表的表名，<font color='red'>注意:这里有时候是中间结果表的表名，MySQL会按照自己的规则生成</font>
##### *type*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;type的取值在很大的程度上反应了SQL的执行性能。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;按照性能由高到底，type的取值依次为：NULL，system，const，eq_reg，ref，range，index，ALL
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*NULL 不用查表，速度最快*
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_null.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*system 当表中只有一条数据的时候 type为system*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*const 常数查询 一般是根据唯一键或者主键等值查询*
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explain_const.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*eq_reg 表连接的时候 在b表查询出来的结果在a表这中按照唯一索引值查询一行*
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_eq_ref.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*reg 非唯一索引查询*
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explain_ref.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*range 使用唯一索引返回扫描*
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_range.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*index 扫描整个索引文件，例如覆盖索引的查询，效率只是比全表查询略快，因为索引文件一般比数据文件小，所以一次读入内存的索引数据更多，这样磁盘IO就会更少*
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explain_full_index.png)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*All 表示全表扫描，是效率最低的一种查询*
##### *possible key*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表示可能使用的索引，显示的顺序与表连接的顺序无关。
##### *key*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表示MySQL执行本条sql选的索引的名字，可以通过force idex 和 ignore index 来强制改变sql执行所需要的索引。
##### *key_len*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表示该条索引的占用的自己树，是根据索引字段的类型计算出来的，
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如:int(11)索引长度是4
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;varchar(128)并且编码是U8，索引长度的计算方法为 ： 128*3+2 
##### *ref*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表示使用哪个列从表中选择行，取值有科恩个是const
##### *rows*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表示执行该条SQL必须扫描的行数
##### *extra*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;包含了MySQL生成执行计划的详细信息：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;distinct 查找唯一值，一旦找到就不在继续查找了（暂时没有想好例子）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;record 没有找到理想的索引
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;use file sort 使用外排来排序  效率比较低
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;use index 使用覆盖索引返回数据，没有扫描表
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;use tempoary 使用临时表来组合返回数据 效率较低
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;use where 使用where条件过滤返回的数据，在MySQL的存储引擎层没有过滤完数据，只能在MySQL服务层去过滤数据
#### *profiling详解* 
##### *开启profiling*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为profiling是比较消耗资源的，所以一般的MySQL默认都关闭了profiling功能，并且profiling只是针对当前session有效，目前不支持全局的profiling，可以通过如下的命令查看并开发profiling功能：
 ```sql
 SELECT @@profiling  -- 返回的结果如果是0 表示当前的session的profiling功能是关闭的  
set profiling=1  -- 打开当前session的profiling功能  
 ```
##### *profiling的使用*
###### *查询当前session的profiling的概要信息* 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以使用 show profiles命令获取当前session所执行的sql的概要信息：
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_profiling_show.png)
###### *profiling详解* 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;profiling的语法如下：
```sql
SHOW PROFILE [type [, type] ... ]  
    [FOR QUERY n]  
    [LIMIT row_count [OFFSET offset]]  
type:  
    ALL  
  | BLOCK IO  
  | CONTEXT SWITCHES  
  | CPU  
  | IPC  
  | MEMORY  
  | PAGE FAULTS  
  | SOURCE  
  | SWAPS 
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用示例：
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_profiling.png)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;结果说明：   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在使用profiling查看sql的详细执行计划的时候，主要关注的是前两列即：status和duration
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;status 表示sql的执行状态和 show full process list 查看到的状态一致
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;duration 表示每个状态执行的时间 可以看到sql的主要执行时间消耗在哪里
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其次需要关注的是cup，io,swap的详细信息
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cup表示 cpu的消耗时间
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;swap表示机器的swap情况
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;io表示io的消耗情况
  
## *无效索引*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在很多时候MySQL的表建立了索引，并且在查询条件中也使用了索引进行筛选，但是并不一定会使用到索引，例如下面的几种情况。
### *筛选条件包含了隐式转换*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;下面的例子中，name字段添加了唯一索引，但是name字段的类型是varchar类型的，而筛选添加时int类型，发生了隐式转换，所以走全表扫描。这里比较隐晦。在上周有一个项目分析酒店订单的时候，本来hive中的酒店订单包含了酒店项目的所有订单，订单id是varchar类型的，而我们需要统计QTA中参加某一个活动的订单，需要查询QTA的订单详情库，（QTA订单详情是hive中订单的子集）里面的订单ID是long类型的，最开始查询的时候就直接在一个表查询完后再另外一个表查询，结果看到一条简单的sql执行起来巨慢。最后分析原因就定位到了这个上面。
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explain_convert_all.png)
### *不支持函数式索引*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;age字段上面添加了非唯一索引，但是使用了绝对值函数，所以age字段上面的索引就无法使用了。这个在处理日期的时候经常遇到这样的坑。
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_function_all.png)
### *索引扫描的代价大于直接全表扫描*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果只有索引过滤的数据比较少，那么会直接走全表扫描，因为使用索引的时候会先扫描一遍索引，然后根据扫描到的索引回表找到所需要的数据，这样扫描的效率其实更低，所以直接走全表扫描。
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_all_index.png)
### *使用“%”前缀匹配的时候*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;name字段添加了唯一索引 但是使用‘%’作为前缀匹配条件，所以不使用索引，直接走全表扫描。
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explian_all_like.png)
### *复合索引非左前缀匹配*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在使用复合索引的时候 如果不是使用的左前缀筛选条件 则不会使用索引，还是会全表扫描。
### *or筛选添加前后都有索引的时候才会走索引*
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在使用or作为筛选条件的时候，or的前后筛选条件都必须添加索引 这样才能使用索引 否则 整条sql都无法使用索引。
![](http://7xutce.com1.z0.glb.clouddn.com/mysql_explain_all_or.png)








