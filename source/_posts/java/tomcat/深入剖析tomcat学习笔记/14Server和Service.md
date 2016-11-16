
---
title: Server和Service
date: 2016-11-15 18:11:01
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# Server和Service

## Server接口

Server接口的实例，表示Catalina整个servlet容器，囊括了所有组件。

服务器组件提供了一种非常优雅的方法来启动/关闭整个系统，不再对连接器和容器分别操作：
当启动服务器组件时，它会启动其中所有的组件，然后就无限期地等待关闭命令；
如果需要关闭服务器组件，向指定端口发送一条关闭命令，服务器组件接收到关闭命令后，就会关闭其中的所有组件。

```java
public interface Server {
    // ------------------------------------------------------------- Properties
    // 顶级的全局naming resource
    public NamingResources getGlobalNamingResources();
    public void setGlobalNamingResources(NamingResources globalNamingResources);
        
    // 关闭命令的监听接口
    public int getPort();
    public void setPort(int port);
    
    // 关闭命令字符串
    public String getShutdown();
    public void setShutdown(String shutdown);
    // --------------------------------------------------------- Public Methods
    public void addService(Service service);
    public Service findService(String name);
    public Service[] findServices();
    public void removeService(Service service);
    
    public void await();

    // 启动之前的初始化行为
    public void initialize() throws LifecycleException;
}
```

## StandardServer类

StandardServer类有4个与生命周期有关的方法：initialize()/start()/await()/stop()

### initialize()方法

initialize()方法主要用来初始化其中的服务组件。
```java
public void initialize() throws LifecycleException {
    if (initialized)
        throw new LifecycleException (
            sm.getString("standardServer.initialize.initialized"));
    initialized = true;
    // Initialize our defined Services
    for (int i = 0; i < services.length; i++) {
        services[i].initialize();
    }
}
```

### start()/stop()方法

与initialize()类似，触发相应的事件，循环遍历逐一启动其中service组件。

### await()方法

await()方法负责等待关闭整个tomcat部署的命令。

await()方法创建一个监听默认为8005端口的server socket，循环调用serverSocket.accept()方法等待连接，得到其中字符串时与约定的关闭命令字符串相比较，
如果匹配成功，则跳出循环，结束await()方法。

## Service接口

Service接口实例是一个或多个组成的一组连接器共享一个单独的servlet容器来处理接受到的请求。这样的设定允许了一个http和一个https连接器共享同一个web应用程序。

## StandardService类

### connector和container

StandardService实例中有两种组件，connector和container，其中servlet容器只有一个，而连接器可以多个，使tomcat可以为多种不同的协议提供服务。

StandardService类的属性：
```java
private Container container = null;
private Connector connectors[] = new Connector[0];
```

### 与生命周期有关的方法

initialize()：循环遍历调用了`connectors[i].initialize()`来初始化所有的连接器。

start()/stop()：触发相关事件，并循环遍历启动/关闭其中连接器以及容器。
