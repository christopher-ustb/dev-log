---
title: 记录一次阿里云裸机apt-get使用故障
date: 2016-06-21 20:36:24
categories: 
- Linux
tags:
- Ubuntu
---

# 记录一次阿里云裸机apt-get使用故障

root@host:apt-get install tomcat7

E: unable to locate package tomcat7

root@host:apt-get update

nothing download

/etc/apt/sources.list 文件丢失了

在[askubuntu](http://askubuntu.com/questions/538676/etc-apt-source-list-not-found)找回了一个副本


