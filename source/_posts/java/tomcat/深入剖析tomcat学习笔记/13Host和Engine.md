---
title: Host和Engine
date: 2016-11-14 17:26:48
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# Host和Engine

## Host接口

Host代表了Catalina servlet引擎中的一个虚拟主机。

主要用于以下场景：
* 为指定虚拟主机的每个请求使用拦截器
* 需要将Catalina与一个独立的http连接器配合使用，但是仍然支持多虚拟主机

## StandardHost类

StandardHost类是Catalina中Host接口的标准实现。

StandardHost类的构造器与StandardWrapper/StandardContext类类似，设置StandardHostValve实例作为管道对象的基础阀。
```java
public StandardHost() {
    super();
    pipeline.setBasic(new StandardHostValve());
}
```

StandardHost#start()方法中，为StandardHost添加了两个阀，分别是：
* ErrorReportValve，输出HTML错误页的实现阀
* ErrorDispatcherValve，处理错误分发（即在必要情况下，转发到合适的错误页）的实现阀

类似于StandardContext，StandardHost类并没有重写其父类的invoke()方法，所以直接调用其父类ContainerBase#invoke()方法，
最终调用了其基础阀StandardHostValve#invoke()，第一件事还是寻址：找到合适的Context容器。
最终实现寻址的方法是StandardHost#map(String)：
```java
public Context map(String uri) {

    if (debug > 0)
        log("Mapping request URI '" + uri + "'");
    if (uri == null)
        return (null);

    // Match on the longest possible context path prefix
    if (debug > 1)
        log("  Trying the longest context path prefix");
    Context context = null;
    String mapuri = uri;
    while (true) {
        context = (Context) findChild(mapuri);
        if (context != null)
            break;
        int slash = mapuri.lastIndexOf('/');
        if (slash < 0)
            break;
        mapuri = mapuri.substring(0, slash);
    }

    // If no Context matches, select the default Context
    if (context == null) {
        if (debug > 1)
            log("  Trying the default context");
        context = (Context) findChild("");
    }

    // Complain if no Context has been selected
    if (context == null) {
        log(sm.getString("standardHost.mappingError", uri));
        return (null);
    }

    // Return the mapped Context (if any)
    if (debug > 0)
        log(" Mapped to context '" + context.getPath() + "'");
    return (context);

}
```

## StandardHostValve类

StandardHostValve类是StandardHost实例的基础阀，其invoke()方法主要实现以下处理：
1. 验证request/response对象类型的合法性
2. 选择合适的Context实例来处理请求
3. 将context的类加载器设置为当前线程的classloader
4. 更新session对象的last access time
5. 将request交由context处理

## Engine接口

Engine代表了整个Catalina servlet引擎，当部署Tomcat时需要支持多个虚拟主机时，就需要Engine容器。

实际上，一般情况下部署的tomcat都会使用一个Engine容器。

## StandardEngine类

StandardEngine类是Engine接口的标准实现。

StandardEngine类的构造器还是老规矩，设置一个基础阀：
```java
public StandardEngine() {
    super();
    pipeline.setBasic(new StandardEngineValve());
}
```

## StandardEngineValve类

StandardEngineValve类是StandardEngine实例的基础阀，其invoke()方法主要实现以下处理：
1. 验证request/response对象类型的合法性
2. 验证任意一个HTTP/1.1请求包含一个server_name
3. 选择合适的Host实例来处理请求
4. 将request交由host处理
