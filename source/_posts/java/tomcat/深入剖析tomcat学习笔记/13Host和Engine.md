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


