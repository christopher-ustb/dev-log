---
title: web应用get请求获取中文乱码问题
date: 2016-4-21 14:54:24
categories: 
- Java
tags:
- java
- tomcat
---

# web应用get请求获取中文乱码问题

* spring配置文件applicationSettings.xml添加节点

```xml
<bean
  class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
  <property name="messageConverters">
   <list>
    <bean
     class="org.springframework.http.converter.StringHttpMessageConverter">
     <property name="supportedMediaTypes">
      <list>
       <value>text/plain;charset=UTF-8</value>
      </list>
     </property>
    </bean>
   </list>
  </property>
 </bean>
```


* tomcat配置conf/server.xml添加节点规定容器的url编码

```xml
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" URIEncoding="UTF-8"/>
```