---
title: java和Spring发送邮件
toc: true
date: 2016-05-30 10:04:55
categories: Java
tags:  
- Java
- 邮件
description: 在我们的项目中，很多时候需要用到发送邮件的地方。网上也有一些发送邮件的教程。但是由于有的作者试图使发送邮件的系统比较完善。所以写的比较“曲折”，以至于让读者看到之后不知所云。本文将提炼出发送邮件的骨架。如果读者按照本文的内容开发自己的邮件系统，只需要简单的六步即可。
---

在我们的项目中，很多时候需要用到发送邮件的地方。网上也有一些发送邮件的教程。但是由于有的作者试图使发送邮件的系统比较完善。所以写的比较“曲折”，以至于让读者看到之后不知所云。本文将提炼出发送邮件的骨架。如果读者按照本文的内容开发自己的邮件系统，只需要如下六步即可：
## POM中添加如下的依赖
```xml
<dependency>  
     <groupId>javax.mail</groupId>  
     <artifactId>mail</artifactId>  
 <dependency>  
```
## 在项目的Resource目录中配置发送邮件的服务器信息
配置文件的名称为：mail.properties
配置文件的内容如下：
```xml
mail.host=xxx.xxx.xxx.com  
mail.smtp.auth=false  
mail.smtp.timeout=25000  
mail.from=chenglai.guo@xxxx.com  
  
mail.to=xxxx@xxx.com  
mail.cc=lori.zhang@xxx.com,kai.shao@xxx.co  
```
## 添加spring邮件配置信息
配置文件的名称为：app-mail.xml
配置文件的内容如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:context="http://www.springframework.org/schema/context"  
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd  
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.1.xsd">  
  
    <!--system.properties文件中存有所需的信息-->  
    <context:property-placeholder location="classpath:mail.properties" ignore-unresolvable="true"/>  
  
    <!--JavaMailSender的设定-->  
    <bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">  
        <property name="host" value="${mail.host}"/>  
        <property name="defaultEncoding" value="UTF-8"></property>  
        <property name="javaMailProperties">  
            <props>  
                <prop key="mail.smtp.auth">${mail.smtp.auth}</prop>  
                <prop key="mail.smtp.timeout">${mail.smtp.timeout}</prop>  
            </props>  
        </property>  
    </bean>  
  
     <bean id="mailEntity" class="com.qunar.des.rank.model.mail.MailEntity">  
         <property name="from" value="${mail.from}"/>  
         <property name="to" value="${mail.to}"/>  
         <property name="cc" value="${mail.cc}"/>  
     </bean>  
</beans> 
```
## 在根spring配置文件中引入改文件
```
<import resource="app-mail.xml"/> 
```
## 编写一个简单的发送邮件的类MailServiceImpl.java
```java
package com.qunar.des.rank.service.impl;  
  
import com.qunar.des.rank.model.mail.MailEntity;  
import com.qunar.des.rank.service.MailService;  
import javax.mail.MessagingException;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
import org.springframework.mail.javamail.JavaMailSender;  
import org.springframework.mail.javamail.MimeMessageHelper;  
import org.springframework.stereotype.Service;  
import javax.annotation.Resource;  
  
/** 
 * Created by chenglaiguo on 12/15/14. 
 */  
@Service("mailService")  
public class MailServiceImpl implements MailService {  
    private Logger logger = LoggerFactory.getLogger(MailServiceImpl.class);  
  
    @Resource  
    private JavaMailSender mailSender;  
    @Resource(name = "mailEntity")  
    private MailEntity mailEntity;  
  
    public boolean send(String subject, String content) {  
        boolean flag = false;  
        MimeMessageHelper mm = this.initMessageHelper(subject, content);  
        if (mm==null){  
            logger.error("MailServiceImpl class send method send mail failure");  
            return flag;  
        }  
        mailSender.send(mm.getMimeMessage());  
        logger.info("MailServiceImpl class send method send mail success");  
        flag = true;  
        return flag;  
    }  
  
    private MimeMessageHelper initMessageHelper(String subject, String content) {  
        MimeMessageHelper mm = null;  
        try {  
            mm = new MimeMessageHelper(mailSender.createMimeMessage(), true, "UTF-8");  
            mm.setFrom(mailEntity.getFrom());  
            mm.setSubject(subject);// 设置邮件主题  
            mm.setText(content);// 生成的邮件内容  
            mm.setTo(mailEntity.getTo());// 发送列表  
            mm.setCc(mailEntity.getCc().split(","));// 抄送列表  
        } catch (MessagingException e) {  
            logger.error("MailServiceImpl class initMessageHelper method cause MessagingException : ",e);  
        }  
        return mm;  
    }  
}  
```
## 编写一个测试类
```java
package com.qunar.des.rank.service;  
  
import com.qunar.des.rank.TestBase;  
import com.qunar.des.rank.service.impl.MailServiceImpl;  
import org.junit.Test;  
import org.springframework.beans.factory.annotation.Autowired;  
import static org.junit.Assert.assertTrue;  
  
/** 
 * Created by chenglaiguo on 12/15/14. 
 */  
public class MailServiceTest extends TestBase {  
    @Autowired  
    private MailServiceImpl mailService;  
  
    @Test  
    public void testSendMail() {  
        boolean flag = false;  
        flag = mailService.send("test","test2");  
        assertTrue(flag);  
    }  
  
}  
```
至此一个简单的发送邮件的系统已经开发完成了。很简单吧！！！
附件：MailEntity.java
```java
package com.qunar.des.rank.model.mail.MailEntity;  
/** 
 * Created by guochenglai on 11/17/15. 
 */  
public class  MailEntity {  
    private String from ="";  
    private String to="";  
    private String cc = "";  
  
    public String getFrom() {  
        return from;  
    }  
  
    public void setFrom(String from) {  
        this.from = from;  
    }  
  
    public String getTo() {  
        return to;  
    }  
  
    public void setTo(String to) {  
        this.to = to;  
    }  
  
    public String getCc() {  
        return cc;  
    }  
  
    public void setCc(String cc) {  
        this.cc = cc;  
    }  
} 
```