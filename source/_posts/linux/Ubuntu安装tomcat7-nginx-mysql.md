---
title: Ubuntu安装tomcat7 nginx mysql等服务
date: 2016-03-04 21:36:24
categories: 
- Linux
tags:
- 部署
---

# Ubuntu运用apt-get install tomcat7 nginx mysql等服务

## apt-get install tomcat7

tomcat7包依赖于JDK，如果没有安装JDK会自动安装OpenJDK
```
java version "1.7.0_101"
OpenJDK Runtime Environment (IcedTea 2.6.6) (7u101-2.6.6-0ubuntu0.14.04.1)
OpenJDK 64-Bit Server VM (build 24.95-b01, mixed mode)
```

CATALINA_HOME  /usr/share/tomcat7
CATALINA_BASE /var/lib/tomcat7

## apt-get install nginx

低版本nginx不支持always关键字，但是Ubuntu的中央repository最新nginx版本是1.4.6，
必须添加[PPA](https://en.wikipedia.org/wiki/Personal_Package_Archive)更新nginx版本。

```bash
# 安装software-properties-common管理Ubuntu软件应用安装来源信任，允许管理分布式和个人软件资源。
$ apt-get install software-properties-common
$ apt-get install python-software-properties
# 将nginx/stable的ppa加入信任
$ add-apt-repository ppa:nginx/stable
# 更新package list
$ apt-get update
# 升级或安装nginx
$ apt-get upgrade nginx/apt-get install nginx
```

nginx所有配置在目录：/etc/nginx/
日志文件目录：/var/log/nginx/

* 修改nginx最大body大小解决413 Request Entity Too Large错误

	nginx.cnf中http{}节点中增加配置

	```
		client_max_body_size 2m;
	```



