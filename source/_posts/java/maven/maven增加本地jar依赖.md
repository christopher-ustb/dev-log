---
title: maven增加本地jar包依赖
date: 2016-8-13 19:54:24
categories: 
- Java
tags:
- java
- maven
---

# 如何为maven增加本地jar包的依赖

## 起因
在接入腾讯开放平台过程中，发现腾讯的SDK并没有增加在maven的中央仓库，也没有开放源代码，只是发布了一个jar包。

所以需要将这个完全本地的jar包加入到maven的dependencies列表中。

## 错误的示范

简单google之后，试用了这种方案：
```xml
<dependency>
    <groupId>sample</groupId>
    <artifactId>com.sample</artifactId>
    <version>1.0</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/src/main/resources/yourJar.jar</systemPath>
</dependency>
```
将此依赖指定为system依赖，然后指定对应的系统路径。

将此module打包也能成功。

虽然这个方案看上去十分简单，却是错的。

但是由于我在使用一个multiple modules project，webapp module作为最后的release模块，依赖于其他许多子模块。

scope=system的dependency被认定为系统级依赖，不会被复制到最终的应用的war包里。

## [将未经maven管理的依赖引入项目](https://devcenter.heroku.com/articles/local-maven-dependencies)

1. 选定groupId, artifactId, version

    这些参数对用户来说毫不重要，但是maven对所有的dependency都需要这些信息。
    
    所以此处姑且使用:
    
    * groupId: com.example
    * artifactId: mylib
    * version: 1.0

2. 创建一个本地maven依赖仓库目录

    在module根目录下添加一个`repo`目录（与pom.xml平级）

3. 将jar作为artifact安装进repo

    执行maven goal:
    
    ```
    mvn deploy:deploy-file -Durl=file:///path/to/yourproject/repo/ -Dfile=mylib.jar -DgroupId=com.example -DartifactId=mylib -Dpackaging=jar -Dversion=1.0
    ```
    执行成功后，repo目录会产生一些包含pom文件等元数据以及jar包文件的目录。

4. 为当前module增加本地仓库

    ```xml
     <!--other repositories if any-->
    <repository>
        <id>project.local</id>
        <name>project</name>
        <url>file:${project.basedir}/repo</url>
    </repository>
    ```

5. 为当前module增加本地依赖
    ```xml
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>mylib</artifactId>
        <version>1.0</version>
    </dependency>
    ```

---
此时再进行打包，会发现mylib.jar变成了mylib-1.0.jar，并且会像其他jar包一样会复制到target的lib目录

