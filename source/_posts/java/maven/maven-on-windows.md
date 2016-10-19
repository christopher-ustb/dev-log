---
title: windows安装maven
date: 2016-08-01 14:04:24
categories: 
- Java
tags:
- java
- maven
---

1. 安装jdk及配置环境变量

2. 安装maven，并配置PATH

3. 修改本地仓库地址
    
    %M2_HOME%\conf\settings.xml搜索localRepository节点
    
    注意：此处File.separator要填入正斜杠，即使是windows系统

4. 修改JDK版本

    在profiles节点增加

    ```xml
    <profile>       
           <id>jdk-1.7</id>       
           <activation>       
               <activeByDefault>true</activeByDefault>       
               <jdk>1.7</jdk>       
           </activation>       
           <properties>       
               <maven.compiler.source>1.7</maven.compiler.source>       
               <maven.compiler.target>1.7</maven.compiler.target>       
               <maven.compiler.compilerVersion>1.7</maven.compiler.compilerVersion>       
           </properties>       
   </profile>
    ```
    
5. 修改远程maven仓库镜像地址（针对访问外网较慢的环境）
    
    在mirrors节点增加（oschina第三方仓库，服务不是很稳定，但速度较快）
    ```xml
    <mirror>
        <id>nexus-osc-thirdparty</id>
        <mirrorOf>thirdparty</mirrorOf>
        <name>Nexus osc thirdparty</name>
        <url>http://maven.oschina.net/content/repositories/thirdparty/</url>
    </mirror>
    ```