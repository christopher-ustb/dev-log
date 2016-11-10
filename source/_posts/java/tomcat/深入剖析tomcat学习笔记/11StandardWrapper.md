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



