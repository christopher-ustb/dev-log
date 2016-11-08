---
title: servlet容器
date: 2016-11-4 16:51:41
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# servlet容器

## Container接口

Catalina中的servlet容器，共有4种类型：
* Engine： 表示整个Catalina servlet引擎
* Host： 表示包含多个Context容器的虚拟主机
* Context： 表示一个包含多个wrapper的web应用程序
* Wrapper： 表示一个独立的servlet

可以看出，“容器”这个命名还是比较通俗的，因为每个容器都在盛装东西嘛。

由于不同级别之间container是一种类似树形结构的关系，所以会有一些相关的方法：

```java
Container findChild(String name);
Container[] findChildren();
void addChild(Container child);
void removeChild(Container child);
```

容器的设计上会包含一些组件化的功能，所以也定义了一些属性，如：
* Loader： 加载容器中新的类的classloader
* Logger： 日志记录器
* Manager： session池管理器
* Realm： 只读的安全领域接口，做身份验证
* Resources： 静态资源访问器

## 管道任务

管道包含了servlet容器所要调用的任务，一个阀表示一个具体的执行任务，阀都装在管道上。
这个模型的比喻其实相当好理解，也会让人不禁想到了责任链模式，让人不禁想到了servlet编程中的filterChain过滤器链。

一个Pipeline有多个Valve，但是只有一个会被标记为BasicValve。

tomcat的设计者没有使用一个循环遍历Valve数组的方式来执行任务，而是使用了一个ValveContext来实现责任链模式的遍历执行。

### Pipeline接口

描述了一个应该顺序执行invoke()方法的Valve集合，通常每个容器都绑定一个Pipeline实例，并且容器的请求处理功能点是封装在管道内部的特定阀，通常将最基础的功能定义为Basic阀，在管道的最后执行。
其他的阀会按照它们被添加的顺序依次执行。
接口主要定义了增加减少阀、设置查找Basic阀以及invoke

```java
public interface Pipeline {
    Valve getBasic();
    void setBasic(Valve valve);
    void addValve(Valve valve);
    Valve[] getValves();
    void removeValve(Valve valve);
    void invoke(Request request, Response response)throws IOException, ServletException;
}
```

### Valve接口

```java
public interface Valve {
    void invoke(Request request, Response response, ValveContext context);
}
```

### ValveContext接口

描述了管道内一个阀能够触发下一个阀的执行的机制，而无需知道其内部的实现细节（上文所提到的循环遍历数据就是其中一种实现细节）。

```java
public interface ValveContext {
    void invokeNext(Request request, Response response);
}
```

### Contained接口

一个解耦一个class必须绑定至多一个container实例的接口，描述这个类是带有容器的。

```java
public interface Contained {
    Container getContainer();
    void setContainer(Container container);
}
```

## Wrapper

wrapper级别容器代表一个web应用部署后独立的servlet定义，并且其实现负责管理该servlet的生命周期。

## Context

大部分的web应用中，需要多个servlet合作，这时需要的servlet容器是Context，而不是Wrapper。
