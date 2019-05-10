---
title: Java读取jar包中文件夹内容
date: 2019-5-10 09:56:08
categories:
- Java
tags:
- Java
---

# Java读取jar包中文件夹内容

## 目标

运行时列举/src/main/resources/configs目录下的所有文件

## 错误示例

```java
File f = new File(this.getClass().getResource("/configs/").getPath());

for(String s : f.list){
   System.out.println(s);
}
```

在IDE或者mvn spring-boot:run时，可以工作。但是打jar包后，使用java -jar命令运行时，找不到这个目录。

文件被打包进jar之后，当然不存在/configs/这个目录，再用java.io.File去取就是缘木求鱼了。

## 解决方案

在spring框架中，对这个问题提供了比较优雅的API：

```java
PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();

Resource[] resources = resolver.getResources("classpath:/configs/*.*");
```

`org.springframework.core.io.support.PathMatchingResourcePatternResolver`这个类取自spring-core包，通过传入getResources()方法的父级目录，来判断路径协议是`file://`还是`jar://`:

```java
else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) {
    result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
}
else {
    result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
}
```

其中，`doFindPathMatchingJarResources`,`doFindPathMatchingFileResources`方法分别实现了在jar中定位resources（利用`java.util.jar.JarFile`类）、在file系统中定位resources的功能。

## 总结

spring这个工具类同时支持了ide中运行和jar运行，节省了开发很多兼容问题的代码，是一个很好的实现。
