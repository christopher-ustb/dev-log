---
title: IOC容器之根接口BeanFactory
date: 2017-3-16
categories: 
- Java
tags:
- java
- spring
- 读书笔记
---

# IOC容器之根接口BeanFactory

## BeanFactory

BeanFactory提供了ioc容器最基础的访问接口，可以说是spring框架庞大的ioc体系的树根。

BeanFactory接口规定了其实现必须具有以下特性：

* 装载了一定数量的bean定义，每个定义以一个独一无二的spring name来标识。

* bean定义了某种scope:Singleton/Prototype

	* singleton: 对单例模式的优化，实例在factory范围内唯一

    * prototype: 原型模式，每次返回一个独立的实例

BeanFactory乃至其背后的ioc思想，本质上代表了应用组件的中心化注册、配置与管理，采用依赖注入的Push模式，而不是类似于主动查找的Pull模式。

BeanFactory的方法十分简单，主要分为以下几组：

* 根据名称、类型获取bean，包括一些简单的泛型重载

* 查询一个bean的存在与否/scope/type/alias

## HierarchicalBeanFactory

HierarchicalBeanFactory为BeanFactory扩展了层级特性，提现在两个增加的方法定义上：

```java
BeanFactory getParentBeanFactory();
boolean containsLocalBean(String name);
```

## SingletonBeanRegistry

定义了单例bean的registry，通常被BeanFactory实例所实现，暴露出统一的单例bean管理行为。
主要包括注册singleton、获取singleton等系列方法。

## ConfigurableBeanFactory

看过HierarchicalBeanFactory的定义，很奇怪，为什么只有getParentBeanFactory方法，没有set。
spring开发者将set方法放在HierarchicalBeanFactory的子接口ConfigurableBeanFactory中，ConfigurableBeanFactory同时extends SingletonBeanRegistry。
为什么没有直接将setParentBeanFactory放在HierarchicalBeanFactory中呢？