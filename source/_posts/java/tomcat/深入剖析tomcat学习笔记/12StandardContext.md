---
title: StandardContext
date: 2016-11-11 15:30:26
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# StandardContext

Context实例表示一个具体的Web应用，其中包含多个Wrapper，每个Wrapper表示一个具体的servlet定义。
StandardContext是对Context接口的标准实现。

## StandardContext的配置

### StandardContext类的构造函数

```java
public StandardContext() {
    super();
    pipeline.setBasic(new StandardContextValve());
    namingResources.setContainer(this);
}
```
主要是将StandardContextValve类型的实例设置成了自身管道任务的基础阀，并且将自身的namingResources与自身绑定。

### 启动StandardContext实例

StandardContext实现了Lifecycle接口，方便使用父子关系组件统一启动、停止。

查看start()方法主要完成了以下工作：
1. 触发before_start事件
2. setAvailable(false)
3. setConfigured(false)
4. 配置资源
5. 设置载入器
6. 设置Session管理器
7. 初始化字符集映射器
8. 启动与容器相关的子组件，如载入器、日志记录器、集群支持、领域、资源管理器、寻址器(Mapper)等
9. 启动子容器
10. 启动管道Pipeline
11. 触发start事件
12. 启动Session管理器
13. 如果启动成功，创建context属性，loadOnStartup(findChildren())，载入启动时就载入的子容器，即Wrapper容器
14. 触发after_start事件

### invoke()方法

StandardContext#invoke()方法比较简单，先检查应用程序是否在重载中，若是，等待重载完成。
然后调用父类ContainerBase#invoke()，即`pipeline.invoke();`。

## StandardContextMapper类

StandardContext管道对象的基础阀是StandardContextValve，其invoke()方法实现了StandardContext的基础功能。

StandardContextValve#invoke()的第一件事，就是获取一个要处理HTTP请求Wrapper实例，而负责这个查找工作的组件就是Mapper映射器。

StandardContext#start()方法中，调用了其父类`addDefaultMapper(this.mapperClass);`方法来添加Mapper对象，
而StandardContext定义的`private String mapperClass = "org.apache.catalina.core.StandardContextMapper";`，将StandardContextMapper类设置成了StandardContext的默认映射器。

Mapper接口最重要的方法map()签名如下：
```java
Container map(Request request, boolean update);
```

StandardContextMapper#map()先标识出相对于Context的URL，然后试图应用匹配规则找到一个合适的Wrapper实例。

查看源代码可知，具体的匹配规则按照优先级从高到低有如下：

1. 精确匹配
2. 前缀匹配
3. 后缀扩展名匹配
4. 默认匹配

## 对重载的支持

StandardContext类定义了reloadable属性来指明该应用是否启用了重载功能。
当启用了重载功能后，当web.xml发生变化或WEB-INF/classes下的一个文件被重新编译，应用将会重载。

StandardContext类是通过其载入器实现应用的重载的，tomcat4中，StandardContext对象在的属性WebappLoader类（implements Runnable）实现了Loader接口，并使用另一个线程检查对应目录下所有类和JAR文件的时间戳。

## backgroundProcess()方法

Context容器需要许多组件的支持，这些组件需要使用各自的线程执行一些后台处理任务，如载入器的周期性检查时间戳来实现自动重载、Session管理器的周期性检查来失效Session对象等。

在tomcat4中，这些组件各自拥有线程。在tomcat5中，为了节省资源，所有的后台处理共享一个线程。

如果某个组件或者servlet容器需要周期性地执行一个操作，只需要将其写到backgroundProcess()方法中即可。
