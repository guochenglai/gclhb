---
title: 使用try-with-resource改进代码异常处理机制
toc: true
mathjax: true
date: 2016-06-01 20:48:54
categories: Java
tags: 
- Java 
- 异常处理
description: JDK1.7中引入了try-with-resource机制，试图简化异常处理以及资源关闭等问题。由try-with-resource语句托管的资源文件，在离开try-with-resource语句块的时候，会依次进行关闭。而不再需要像以前那样需要逐个的try-catch并关闭资源。
---
# JDK1.7之前标准的异常处理机制
  在JAVA7之前，程序中必须顺次打开或者关闭资源，如果只打开了资源没有关闭资源。就会出现资源泄漏问题，线上代码运行时间越久，程序的效率就会越低。但是，资源的关闭不仅繁琐而且很容易出问题。
## 常见的关闭资源的问题。
在如下关闭资源的finally语句中。如果在关闭resultSet对象的时候出现了异常，则后面的prepareStatement和connection对象都没法正常关闭。 
```java
package com.qunar.des.baofang.common;  
  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
  
import java.sql.*;  
  
/** 
 * Created by guochenglai on 5/24/16. 
 */  
public class MainClass {  
    private static final String DRIVER_CLASS = "com.mysql.jdbc.Driver";  
    private static final String URL = "jdbc:mysql://127.0.0.1:3306/test" ;  
    private static final String USER_NAME = "test" ;  
    private static final String PASSWORD = "test" ;  
    private static final String SQL = "select * from test limit 1";  
    private static final Logger LOGGER = LoggerFactory.getLogger(MainClass.class);  
    public static void main(String[] args) {  
        testExecuteQueryByOld();  
    }  
  
    public static void testExecuteQueryByOld() {  
        try {  
            Class.forName(DRIVER_CLASS);  
        } catch (ClassNotFoundException e) {  
            LOGGER.error("can not find driver class",e);  
            return;  
        }  
        Connection connection = null;  
        PreparedStatement preparedStatement = null;  
        ResultSet resultSet = null;  
        try {  
            connection = DriverManager.getConnection(URL, USER_NAME, PASSWORD);  
            preparedStatement = connection.prepareStatement(SQL);  
            resultSet = preparedStatement.executeQuery();  
            if (resultSet.next()) {  
                System.out.println(resultSet.getObject(1)+" : "+resultSet.getObject(2));  
            }  
        } catch (SQLException e) {  
            LOGGER.error("execute sql cause exception  ",e);  
        }finally {  
            try {  
                if (resultSet != null) {  
                    resultSet.close();   
                }  
                if (preparedStatement != null) {  
                    preparedStatement.close();  
                }  
                if (connection!=null) {  
                    connection.close();  
                }  
            } catch (Exception e) {  
                    LOGGER.error("close resource cause exception ",e);  
                }  
            }  
  
        }  
}  
```
## 正常关闭资源方法 
如下的代码块可以认为是一个标准的关闭资源的方法。不论在哪里发生了异常，能够保证所有的资源正确关闭。
```java
package com.qunar.des.baofang.common;  
  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
  
import java.sql.*;  
  
/** 
 * Created by guochenglai on 5/24/16. 
 */  
public class MainClass {  
    private static final String DRIVER_CLASS = "com.mysql.jdbc.Driver";  
    private static final String URL = "jdbc:mysql://127.0.0.1:3306/test" ;  
    private static final String USER_NAME = "test" ;  
    private static final String PASSWORD = "test" ;  
    private static final String SQL = "select * from test limit 1";  
    private static final Logger LOGGER = LoggerFactory.getLogger(MainClass.class);  
    public static void main(String[] args) {  
        testExecuteQueryByOld();  
    }  
  
    public static void testExecuteQueryByOld() {  
        try {  
            Class.forName(DRIVER_CLASS);  
        } catch (ClassNotFoundException e) {  
            LOGGER.error("can not find driver class",e);  
            return;  
        }  
        Connection connection = null;  
        PreparedStatement preparedStatement = null;  
        ResultSet resultSet = null;  
        try {  
            connection = DriverManager.getConnection(URL, USER_NAME, PASSWORD);  
            preparedStatement = connection.prepareStatement(SQL);  
            resultSet = preparedStatement.executeQuery();  
            if (resultSet.next()) {  
                System.out.println(resultSet.getObject(1)+" : "+resultSet.getObject(2));  
            }  
        } catch (SQLException e) {  
            LOGGER.error("execute sql cause exception  ",e);  
        }finally {  
            if (resultSet != null) {  
                try {  
                    resultSet.close();  
                } catch (Exception e) {  
                    LOGGER.error("close result cause exception ",e);  
                }  
            }  
            if (preparedStatement != null) {  
                try {  
                    preparedStatement.close();  
                } catch (Exception e) {  
                    LOGGER.error("close preparedStatement cause exception ",e);  
                }  
            }  
            if (connection != null) {  
                try {  
                    connection.close();  
                } catch (Exception e) {  
                    LOGGER.error("close connection cause exception ",e);  
                }  
            }  
        }  
    }  
}  
```
# 使用try-with-resource改进异常代码的处理机制
JDK1.7中引入了try-with-resource机制，试图简化异常处理以及资源关闭等问题。由try-with-resource语句托管的资源文件，在离开try-with-resource语句块的时候，会依次进行关闭，实现的方法和1.2中的方法基本一样。但是从语法层面上看，不再需要程序员手动去关闭资源，使得程序更加简洁。
## try-with-resource语句示例
```java
package com.qunar.des.baofang.common;  
  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
  
import java.sql.*;  
  
/** 
 * Created by guochenglai on 5/24/16. 
 */  
public class MainClass {  
    private static final String DRIVER_CLASS = "com.mysql.jdbc.Driver";  
    private static final String URL = "jdbc:mysql://127.0.0.1:3306/test" ;  
    private static final String USER_NAME = "test" ;  
    private static final String PASSWORD = "test" ;  
    private static final String SQL = "select * from test limit 1";  
    private static final Logger LOGGER = LoggerFactory.getLogger(MainClass.class);  
    public static void main(String[] args) {  
        testExecuteQueryNew();  
    }  
  
    /** 
     * 使用try-with-resources块改进异常处理代码 
     */  
    public static void testExecuteQueryNew() {  
        try {  
            Class.forName(DRIVER_CLASS);  
        } catch (ClassNotFoundException e) {  
            LOGGER.error("can not find driver class",e);  
            return;  
        }  
        try (Connection connection = DriverManager.getConnection(URL, USER_NAME, PASSWORD); PreparedStatement preparedStatement = connection.prepareStatement(SQL); ResultSet resultSet = preparedStatement.executeQuery();) {  
  
            if (resultSet.next()) {  
                System.out.println(resultSet.getObject(1)+" : "+resultSet.getObject(2));  
            }  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
}  
```
使用try-with-resource方法之后有如下两个好处：
- 代码变得简洁可读
- 所有的资源都托管给try-with-resource语句，能够保证所有的资源被正确关闭，再也不用担心资源关闭的问题。