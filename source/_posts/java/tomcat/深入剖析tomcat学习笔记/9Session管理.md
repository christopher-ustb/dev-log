---
title: Session管理
date: 2016-11-9 14:17:12
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# Session管理

Catalina通过一个称为Session管理器的组件来管理建立Session对象，当请求到来时，要可以返回一个有效的session对象。

## Session对象

这里还是一个老生常谈的外观模式，JEE要求的Session接口是javax.servlet.http.HttpSession，Catalina内部的外观接口是Session，标准实现是StandardSession，用一个StandardSessionFacade的外观包装类来掩盖掉具体实现类的细节。

## Manager

### Manager接口

Manager管理一个关联了特定容器的session池，定义了一些创建、销毁、失效检查、持久化与读取的行为。
不同的Manager实现可能会支持一些不同的附加特性，如session数据持久化、分布式应用服务器的session迁移等。

### ManagerBase抽象类

所有的Session管理器组件都会继承此类，此类提供了createSession(),add(Session),remove(Session),findSession(String)等诸多功能。

### StandardManager类

StandardManager类是Manager接口的标准实现，该类将Session存储在内存中。
StandardManager类实现了Lifecycle接口，这样可以由与其关联的Context容器来启动和关闭。
其中stop()方法的实现会调用unload()方法，以便将有效的session对象序列化到一个名为SESSION.ser的文件中（该文件在$Catalina_home/work目录下）。
当StandardManager实例再次启动，会调用load()方法将这些对象重新读入内存中。

StandardManager类还会负责销毁那些已经失效的Session对象，在tomcat4中，这项工作是由一个专门的线程来完成的。详见StandardManager类的run()方法：
```java
/**
 * The background thread that checks for session timeouts and shutdown.
 */
public void run() {
    // Loop until the termination semaphore is set
    while (!threadDone) {
        threadSleep();
        processExpires();
    }
}
```

### PersistentManagerBase抽象类

PersistentManagerBase类是所有持久化Session管理器的父类。
StandardManager类和持久化Session管理器的区别在于后者中存储器的表现形式，即存储Session对象的辅助存储器的形式，使用Store接口的对象来表示。

在持久化Session管理器中，Session对象可以备份，也可以换出，节省了内存空间。

在tomcat4中，PersistentManagerBase抽象类实现了Runnable接口，使用一个专门的线程来执行备份与换出活动的Session对象的任务。
```java
/**
 * The background thread that checks for session timeouts and shutdown.
 */
public void run() {
    // Loop until the termination semaphore is set
    while (!threadDone) {
        threadSleep();
        processExpires();
        processPersistenceChecks();
    }
}
/**
 * Called by the background thread after active sessions have
 * been checked for expiration, to allow sessions to be
 * swapped out, backed up, etc.
 */
public void processPersistenceChecks() {
    // 换出空闲时间过长的session到辅助存储器中
    processMaxIdleSwaps();
    // 当活动session过多时换出部分相对空闲的session到辅助存储器中
    processMaxActiveSwaps();
    // 备份空闲session
    processMaxIdleBackups();
}
```

### PersistentManager类

继承自PersistentManagerBase抽象类，没有添加或重新新的方法，只是改变了自身的info属性。

### DistributedManager类

DistributedManager类用于2个或多个节点的集群环境，来支持复制Session对象。
为了实现复制Session的目的，DistributedManager类中创建或销毁Session对象时，会向其他节点发送消息，所以集群中的每个节点也必须能够接收其他节点的消息。
集群间的发送和接收行为，分别被定义成了ClusterSender/ClusterReceiver接口。

## 存储器

存储器是org.apache.catalina.Store接口的实例，是为Session管理器管理的Session对象提供持久化存储的一个组件。

```java
public interface Store {
    // 查询所有的session_id
    String[] keys();
    // 载入指定session_id的session
    Session load(String id);
    // 移除指定session_id的session
    void remove(String id);
    // 持久化指定session_id的session
    void save(Session session);
}
```

Catalina提供了几个常用实现类：

### StoreBase抽象类

提供了`processExpires()`功能来销毁过期的session对象，但是load()/save()这些方法都依赖具体的存储媒介，交由子类实现。
    
### FileStore类
    
文件存储媒介是.session的文件

### JDBCStore类

将Session对象通过JDBC存入数据库
