---
title: 一个简单的servlet容器
date: 2016-11-3 14:39:07
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# 一个简单的servlet容器

## javax.servlet.Servlet接口

Servlet接口定义了servlet的基本行为，是JavaEE的标准部分。

主要定义了如下方法：
```java
init(config);
service(request, response);
destroy();
```

## 应用程序

一个功能齐全的servlet容器，应该要处理以下事情：
1. 当第一次调用某个servlet时，载入该servlet类，并调用其init()方法
2. 针对每次请求，创建request/response实例
3. 调用service()方法
4. 当关闭servlet时，调用destroy()方法，并卸载该类

ServletProcessor1类
```java
public class ServletProcessor1{
    public void proccess(Request request, Response response){
        String servletName = request.getUri().substring(request.getUri().lastIndexOf("/") + 1);
        URLClassloader loader = new URLClassloader(new URL[]{
            new URL(null, repository)
        });
        Class clazz = loader.loadClass(servletName);
        Servlet servlet = (Servlet)clazz.newInstance();
        servlet.service(request, response);
    }
}
```

## 一个关于外观模式的说明

在上述的代码实现中，将request/response实例传递给service方法时，相当于出现了向上转型。

而这是一种不安全的做法，了解实现细节的servlet程序员就有可能将这两个实例向下转型并调用其实现方法如request.parse()/response.sendStaticResource()，而这种方法是不应该设置为私有的，因为容器代码可能会调用他们。

相比用默认包级别的访问修饰符，熟悉设计模式的开发者肯定就想到了专门用来隐藏实现细节的设计模式————外观模式。

添加外观类实现ServletRequest接口，但是委托具体的实现类实例的私有属性提供能被调用的方法实现。（懂的人应该看模式名称就懂了，在此不赘述设计模式。）

这样的外观模式在tomcat的整个代码实现中会多次出现。

看到这里，我真的不知道是tomcat的开发者过于学院派，还是开发一个平台、框架、类库的时候，确实需要把使用的开发者当做恶人。
