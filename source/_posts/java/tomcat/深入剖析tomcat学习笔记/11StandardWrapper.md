---
title: StandardWrapper
date: 2016-11-10 10:09:28
categories: 
- Java
tags:
- java
- tomcat
- 读书笔记
---

# StandardWrapper

## 方法调用序列

对于每个引入的http请求，具体的方法调用过程如下：
1. 连接器(Connector)创建request/response对象
2. 连接器调用StandardContext实例的invoke()方法
3. StandardContext实例invoke()方法调用其管道实例的invoke()方法，StandardContext中pipeline的基础阀是StandardContextValve
4. StandardContextValve实例invoke()方法获取相应的Wrapper实例，调用Wrapper实例的invoke()方法
5. StandardWrapper类是Wrapper接口的标准实现，StandardWrapper实例的invoke()会调用其管道对象的invoke()方法，StandardWrapper中pipeline的基础阀是StandardWrapperValve
6. StandardWrapperValve实例的invoke()方法调用Wrapper实例的allocate()方法获取servlet实例
7. allocate()方法调用load()方法载入相应servlet类，若已载入，则无需重复载入
8. load()方法调用servlet实例的init()方法
9. StandardWrapperValve调用servlet实例的service()方法

这一块的源代码相当有意思，主要涉及到接口`Container`、`Pipeline`，这两个接口分别制定了作为容器与管道的标准。

而在这两个接口与标准实现类之间，加入了一个基础抽象类`ContainerBase`，这个类的声明是`public abstract class ContainerBase implements Container, Lifecycle, Pipeline`，
这个抽象类提供了基本每一个Container实现类都需要的通用功能，最典型的就是其invoke()方法：委托一个protected的Pipeline实例的invoke()方法来完成Container的invoke()方法。

而这个实现了Pipeline接口的StandardPipeline实例的invoke()方法，其内部是一个被抽象掉细节的阀invoke()调用链条。

而StandardContext与StandardWrapper都继承自ContainerBase抽象类，并且在自己的构造器中设置Pipeline对象的基础阀，这个基础阀实现了容器最基础的功能。

StandardContextValve是StandardContext的基础阀，除了禁止访问`META-INF`/`WEB-INF`这两个保留目录外，主要功能就是根据请求找到合适的Wrapper实例。而这个寻找的过程，其实就是后端路由的寻址，从面向对象角度来说，这个是不可能靠这个Valve来实现的，于是代码中也是找到关联的Context容器来寻址。

类似的，StandardWrapperValve是StandardWrapper的基础阀，其主要的功能就是让servlet被调用，而具体的调用方法，仍然是交由与其关联的Wrapper容器。

于是，一个精彩的**责任链模式**已经跃然纸上，容器-管道-阀三者之间实现了功能上的组合，但是代码逻辑上的松耦合。

容器的标准实现将自己的核心功能作为基础阀，放在了自己的管道里，但是基础阀主要是对容器API功能的调用来实现要求，并没有将容器的功能搬到阀中。

但是由于管道模型的出现，让开发者有机会在各个级别的容器中增加自己的阀来定制处理流程，而开发者只需要编写自己的阀代码加入管道即可。

## SingleThreadModel

SingleThreadModel接口是一个标记性接口，实现了SingleThreadModel的servlet被称为STM servlet类。

根据Servlet规范，实现此接口的目的，是为了保证servlet实例一次只处理一个请求，即不会出现两个线程同时执行一个servlet实例的service()方法。

其实该接口并不能防止servlet访问共享资源造成的同步问题，因为出现争用的不一定是servlet的实例变量，也有可能是其他类的静态变量或者servlet作用域之外的类，而后者这种情况不胜枚举。

## StandardWrapper类

StandardWrapper对象的主要任务，是载入所需的servlet类，并进行实例化，交由StandardWrapperValve调用servlet的service()方法。

### 分配servlet实例

Wrapper接口定义了allocate()方法：
```java
Servlet allocate() throws ServletException;
```
要求分配一个准备好执行service()方法的已经初始化完成的servlet实例。
如果这个servlet类没有实现SingleThreadModel接口，那么立即返回已初始化的实例；
如果实现了SingleThreadModel接口，Wrapper实现类必须保证，直到调用deallocate()取消这个servlet实例的分配，才重新分配这个servlet实例。

所以StandardWrapper类的allocate()方法主要分为两个部分，分别处理实现STM servlet与非STM servlet。

对于非STM servlet类，或者初次加载servlet类还不知道是否是STM servlet，只是使用了一个`private Servlet instance`的变量来保存这个唯一的servlet实例。
```java
if (instance == null) {
    synchronized (this) {
        if (instance == null) {
            try {
                instance = loadServlet();
            } catch (ServletException e) {
                throw e;
            } catch (Throwable e) {
                throw new ServletException
                    (sm.getString("standardWrapper.allocate"), e);
            }
        }
    }
}
```
在loadServlet()中会判断servlet是否STM。

对于STM servlet类，StandardWrapper会试图从一个对象池中获取可用实例，这个对象池的类型：
```java
/**
 * Stack containing the STM instances.
 */
private Stack instancePool = null;
```
代码与所有的池设计类似，不再赘述。
与之相关的变量有：
* `private int countAllocated = 0;` 当前活跃的分配个数
* `private int nInstances = 0;` 当前已加载的实例个数
* `private int maxInstances = 20;` 对象池最大容量

### 载入servlet类

Wrapper接口声明了load()方法：
```java
void load() throws ServletException;
```
要求加载和初始化一个servlet实例，即载入servlet类，并调用其init()方法。

下面看一下StandardWrapper类loadServlet()方法是如何工作的：
```java
// Nothing to do if we already have an instance or an instance pool
if (!singleThreadModel && (instance != null))
    return instance;
```
如果不是STM Servlet并且实例存在，则直接返回该实例。

```java
if ((actualClass == null) && (jspFile != null)) {
    Wrapper jspWrapper = (Wrapper)
        ((Context) getParent()).findChild(Constants.JSP_SERVLET_NAME);
    if (jspWrapper != null)
        actualClass = jspWrapper.getServletClass();
}
```
如果Servlet是一个JSP页面，那么获取jsp页面对应的实际Servlet类。

```java
// Complain if no servlet class has been specified
if (actualClass == null) {
    unavailable(null);
    throw new ServletException
        (sm.getString("standardWrapper.notClass", getName()));
}
```
如果没有指定class名，抛出异常，终止后续操作。

```java
// Acquire an instance of the class loader to be used
Loader loader = getLoader();
if (loader == null) {
    unavailable(null);
    throw new ServletException
        (sm.getString("standardWrapper.missingLoader", getName()));
}
```
获取载入器，如果找不到载入器，抛出异常。

```java
// Special case class loader for a container provided servlet
if (isContainerProvidedServlet(actualClass)) {
    classLoader = this.getClass().getClassLoader();
    log(sm.getString
          ("standardWrapper.containerServlet", getName()));
}
```
如果servlet是Catalina提供的一些用于访问servlet容器内部数据的专用servlet类，那么将classLoader赋值为容器的ClassLoader实例。

之后是用classloader载入servlet类-实例化-触发相关事件-调用servlet#init()-判断是否是STM servlet-初始化对象池的过程。

## StandardWrapperFacade类

Servlet实例的init()方法需要传入javax.servlet.ServletConfig类型的参数，而StandardWrapper类本身其实是实现了ServletConfig接口的，所以理论上将StandardWrapper本身this传递给servlet就可以了。

但还是老生常谈，这里运用了一个StandardWrapperFacade类作为外观类来隐藏掉StandardWrapper的方法。

## StandardWrapperValve类

StandardWrapperValve类是StandardWrapper实例中的基础阀，完成两个操作：
1. 执行与servlet相关的全部过滤器
2. 调用servlet的service()方法

实际操作步骤如下：
1. 调用StandardWrapper实例allocate()获取servlet实例
2. 调用私有方法createFilterChain()创建过滤器链
3. 调用ApplicationFilterChain#doFilter()执行过滤以及调用service()方法
4. 释放过滤器链
5. 调用Wrapper实例的deallocate()方法取消分配
6. 若该servlet不再被使用到，则调用Wrapper#unload()方法

## FilterDef类

FilterDef表示一个过滤器的定义，正如在部署描述器(web.xml)中定义的\<filter\>节点那样。
```java
public final class FilterDef {
    // 实现这个过滤器的java class全限定名
    private String filterClass = null;
    // 过滤器名称，在特定的web应用中必须是唯一的
    private String filterName = null;
    // 过滤器的初始化参数集合
    private Map parameters = new HashMap();
}
```

## ApplicationFilterConfig类

ApplicationFilterConfig类实现javax.servlet.FilterConfig接口，用于管理Web应用第一次启动时创建的过滤器实例。
```java
public ApplicationFilterConfig(Context context, FilterDef filterDef)
    throws ClassCastException, ClassNotFoundException,
           IllegalAccessException, InstantiationException,
           ServletException {
    super();
    this.context = context;
    setFilterDef(filterDef);
}
```
它的构造函数，接受一个Context对象表示一个Web应用程序，FilterDef表示一个过滤器定义。
ApplicationFilterConfig#getFilter()方法会返回一个javax.servlet.Filter对象，
如果未实例化，则在此方法中定位classloader-加载类-实例化对象-调用Filter#init()方法。

## ApplicationFilterChain类

ApplicationFilterChain类实现javax.servlet.FilterChain接口。
StandardWrapperValve#invoke()方法会创建ApplicationFilterChain实例，并调用其doFilter()方法。
而ApplicationFilterChain#doFilter()的实现中，会调用第一个过滤器Filter#doFilter(ServletRequest, ServletResponse, FilterChain)，并将自身作为第三个参数传入Filter实例，以达到链式调用的目的。
