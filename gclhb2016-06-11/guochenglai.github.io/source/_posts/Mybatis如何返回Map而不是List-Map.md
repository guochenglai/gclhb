---
title: Mybatis如何返回Map而不是List<Map>
toc: true
date: 2016-05-28 21:02:25
categories: Mybatis
tags: 
- Mybatis
- Mybatis返回Map
description: 在使用Mybatis的时候，有时候我们会有这么一种需求：我们希望通过Mybatis查询某一个表返回的结果是一个Map（HashMap），而这个Map的Key是表的一个字段，Value是另一个字段。然而当我们按照Mybatis的做法，指定查询Mapper语句的resultType为map时返回的结果是一个List<Map>。本文的例子将采用一个简单的方法，直接返回map...
---
## 综述
在使用Mybatis的时候，有时候我们会有这么一种需求：我们希望通过Mybatis查询某一个表返回的结果是一个Map（HashMap），而这个Map的Key是表的一个字段，Value是另一个字段。然而当我们按照Mybatis的做法，指定查询Mapper语句的resultType为map时返回的结果是一个List<Map>。本文的例子将采用一个简单的方法，直接返回map。<!-- more -->
     主要思路是：
        1 定义一个注解，这个注解作用于方法之上。
        2 定义一个mybatis的插件，这个插件是一个拦截器，拦截所有的请求，然后判断这个请求方法上面有没有第一步定义的注解。
        3 如果没有第一个定义的注解，则方法继续执行
        4 如果有第一步定义的注解。则表示该方法需要返回一个map，就获取mybatis的返回结果。遍历整个结果将，结果集封装成map返回。
     有如下几个问题需要读者注意：
        因为mybatis没有提供一些内部类的访问接口，所以如下教程的内部类都是根据类名通过反射获取的，如果mybatis升级，并且改变了方法或者类的名称，这个代码就会出现不可预见的问题。不过mybatis升级了这么多版本也没见更改方法名称的改动出现。而且mybatis提供的官方插件也是通过反射裸写方法名获取的，这个问题出现的概率不大。但是如果出现问题，读者应该需要知道这是一个排查点。

## 实现方法
### 注解的定义
   说明：定义注解，只是起一个标识作用，标识该方法需要返回map对象，如果读者有其他的标识方法也是可以的。```java 
package com.qunar.des.baofang.common.interceptor;
import java.lang.annotation.\*;
/**
 * Created by guochenglai on 5/18/16.
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD,ElementType.TYPE})
@Inherited
public @interface MapResult {
    String value();
}```
### 插件（拦截器）的编写
```java
package com.qunar.des.baofang.common.interceptor;

import java.lang.reflect.Method;
import java.sql.ResultSet;
import java.sql.Statement;
import java.util.*;

import org.apache.commons.lang3.StringUtils;
import org.apache.ibatis.executor.resultset.DefaultResultSetHandler;
import org.apache.ibatis.executor.resultset.ResultSetHandler;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;

import com.qunar.des.baofang.common.util.ReflectUtil;

/**
 * Created by guochenglai on 5/18/16.
 */
@Intercepts(@Signature(method="handleResultSets", type=ResultSetHandler.class, args={Statement.class}))
public class MapInterceptor implements Interceptor{

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object target = invocation.getTarget();
        DefaultResultSetHandler defaultResultSetHandler = (DefaultResultSetHandler) target;
        //这里通过反射，根据类名称来获取内部类，如果出现变更，则需要修改，不过mybatis的插件目前都是这样一种情况
        MappedStatement mappedStatement = ReflectUtil.getFieldValue(defaultResultSetHandler, "mappedStatement");

        String className = StringUtils.substringBeforeLast(mappedStatement.getId(), ".");
        String methodName = StringUtils.substringAfterLast(mappedStatement.getId(), ".");

        Method[] methods = Class.forName(className).getDeclaredMethods();
        if (methods == null) {
            return invocation.proceed();
        }
        
        //找到需要执行的method 注意这里是根据方法名称来查找，如果出现方法重载，需要认真考虑
        for (Method method : methods) {
            if (methodName.equalsIgnoreCase(method.getName())) {
                //如果添加了注解标识，就将结果转换成map
                MapResult map = method.getAnnotation(MapResult.class);
                if (map == null) {
                    return invocation.proceed();
                }

                //进行map的转换
                Statement statement = (Statement) invocation.getArgs()[0];

                return result2Map(statement);
            }
        }

        return invocation.proceed();

    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }

    private Object result2Map(Statement statement) throws Throwable{
        ResultSet resultSet = statement.getResultSet();
        if (resultSet == null) {
            return null;
        }

        List<Object> resultList = new ArrayList<Object>();

        Map<Object, Object> map = new HashMap<Object, Object>();

        while (resultSet.next()) {
            map.put(resultSet.getObject(1), resultSet.getObject(2));

        }
        resultList.add(map);
        return resultList;
    }

}
```
### 插件（拦截器）的编写
    需要在数据源中指定我们配置的插件（拦截器）否则我们编写的插件是不起作用的。
```xml
<!-- 数据源的配置这里省略  -->
<!-- session factory -->
    <bean id="reporterSqlSessionFactory" name="biMysqlTuanReportSqlSessionFactory"
          class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:mybatis/reporter-mybatis-config.xml"/>
        <property name="dataSource" ref="appContextDataSource"/>
        <property name="mapperLocations" value="classpath:mapper/reporter/**/*.xml"/>

        <!-- 这里指定刚才编写的插件 -->
        <property name="plugins">
            <array>
                <bean class="com.qunar.des.baofang.common.interceptor.MapInterceptor"/>
            </array>
        </property>
    </bean>
    <!-- mapper scanner configurer -->

    <bean id="reportMapperScannerConfigurer" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.qunar.des.baofang"/>
        <property name="annotationClass" value="com.qunar.des.baofang.common.datasource.DataSource"/>
        <property name="sqlSessionFactoryBeanName" value="reporterSqlSessionFactory"/>
        <property name="nameGenerator" ref="defaultNameGenerator"/>
    </bean>
```
### 在需要返回map的方法上添加注解
   在需要返回map的方法之上添加注解，这样我们的拦截器就能拦截这个方法。并将结果转换成map返回。
```java
package com.qunar.des.baofang.reporter.dao.prepay;

import com.qunar.des.baofang.common.datasource.DataSource;
import com.qunar.des.baofang.common.interceptor.MapResult;
import org.springframework.stereotype.Repository;

import java.util.Map;

/**
 * Created by guochenglai on 5/18/16.
 */
@DataSource("baofangDataSource")
@Repository
public interface PrepayCommentMapper {

    @MapResult("")
    Map<Long, String> queryAllSupplierComment();
}
```
### mapper示例
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.qunar.des.baofang.reporter.dao.prepay.PrepayCommentMapper">

    <select id="queryAllSupplierComment" resultType="map">
        SELECT supplier_id,supplier_comment
        FROM  prepay_supplier_coment
        WHERE is_delete=0
    </select>

</mapper>
```
### 测试示例
```java
package com.qunar.des.baofang.mapper;

import com.alibaba.fastjson.JSON;
import com.qunar.des.baofang.reporter.BaseTester;
import com.qunar.des.baofang.reporter.dao.prepay.PrepayCommentMapper;
import org.junit.Test;

import javax.annotation.Resource;
import java.util.Map;

/**
 * Created by guochenglai on 5/18/16.
 */
public class PrepayCommentMappperTest extends BaseTester{
    @Resource
    private PrepayCommentMapper prepayCommentMapper;

    @Test
    public void testQueryAllSupplierComment() {
        //这里就可以直接以map来接受返回值了
        Map<Long,String> stringMap = prepayCommentMapper.queryAllSupplierComment();
        System.out.println("===============================");
        System.out.println(JSON.toJSONString(stringMap));
        System.out.println("===============================");
    }
}
```
### 输出结果
```
===============================
{2:"dfdf",3:"体育体育"}
===============================
```
### 附件ReflectUtil
```java
package com.qunar.des.baofang.common.util;

import com.qunar.des.baofang.common.model.NumberConstant;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.lang.reflect.Field;
import java.util.Date;

/**
 * Created by guochenglai on 4/19/16.
 */
public class ReflectUtil {

    private static final Logger LOGGER = LoggerFactory.getLogger(ReflectUtil.class);

    /**
     * 根据反射或者一个对象的方法
     * @param obj
     * @param fieldName
     * @param <T>
     * @return
     */
    public static <T> T getFieldValue(Object obj, String fieldName) {
        Object result = null;
        Field field = ReflectUtil.getField(obj, fieldName);
        if (field != null) {
            field.setAccessible(true);
            try {
                result = field.get(obj);
            } catch (IllegalArgumentException e) {
                LOGGER.info("ReflectUtil getFieldValue cause IllegalArgumentException",e);
            } catch (IllegalAccessException e2) {
                LOGGER.info("ReflectUtil getFieldValue cause IllegalAccessException",e2);
            }
        }
        return (T)result;
    }

    private static Field getField(Object obj, String fieldName) {
        Field field = null;
        for (Class<?> clazz=obj.getClass(); clazz != Object.class; clazz=clazz.getSuperclass()) {
            try {
                field = clazz.getDeclaredField(fieldName);
                break;
            } catch (NoSuchFieldException e) {
                LOGGER.info("ReflectUtil getField cause NoSuchFieldException",e);
            }
        }
        return field;
    }

    /**
     * 利用反射 将对象的null中不合理的值设置成null
     * @param object
     */
    public static void setIllegalValueToNull(Object object) {
        if (object == null) {
            return;
        }
        Class<?> clazzType = object.getClass();
        Field[] fields = clazzType.getDeclaredFields();
        try {
            if (fields == null || fields.length == 0) {
                return;
            }

            for (Field field : fields) {
                //double
                field.setAccessible(true);
                if (Double.class.equals(field.getType())) {
                    if ((Double)field.get(object) == NumberConstant.ILLEGAL) {
                        field.set(object, null);
                    }
                }
            }


        } catch (Exception e) {
            LOGGER.error("handle illegal value for target class cause exception ", e);
        }
    }

    /**
     * 利用反射将对象的null值设置成对应类型的默认值
     * @param object
     */
    public static void fillDefaultValue(Object object) {
        if (object == null) {
            return;
        }

        Class<?> clazzType = object.getClass();
        Field[] fields = clazzType.getDeclaredFields();
        try {
            if (fields != null && fields.length > 0) {
                for (Field field : fields) {
                    field.setAccessible(true);
                    Class<?> type = field.getType();
                    String typeName = type.getName();
                    if (field.get(object) == null) {
                        //String
                        if (String.class.getName().equals(typeName)) {
                            field.set(object, "");
                        }
                        //int
                        if (Integer.class.getName().equals(typeName)) {
                            field.set(object, new Integer(0));
                        }
                        //long
                        if (Long.class.getName().equals(typeName)) {
                            field.set(object, new Long(0));
                        }
                        //double
                        if (Double.class.getName().equals(typeName)) {
                            field.set(object, new Double(0.0));
                        }
                        //date
                        if (Date.class.getName().equals(typeName)) {
                            field.set(object, new Date());
                        }
                        //boolean
                        if (Boolean.class.getName().equals(typeName)) {
                            field.set(object, new Boolean(false));
                        }
                        //float
                        if (Float.class.getName().equals(typeName)) {
                            field.set(object,new Float(0));
                        }
                    }
                }
            }
        } catch (Exception e) {
            LOGGER.error("set default value for target class cause exception ", e);
        }
    }

}
```