---
title: windows mysql部署指南
date: 2016-03-02 17:41:24
categories: 
- Mysql
tags:
- windows
- mysql
---

# windows mysql部署指南

1. 解压MySQL免安装包，令其路径为D:\mysql_home

2. 在my-default.ini旁边复制一个my.ini文件，并根据注释修改相关参数

```properties
	[client]
    port=3306
    default-character-set=utf8
    [mysqld]
    port=3306

    character_set_server=utf8

    #解压目录
    basedir=D:\mysql_home
    #解压目录下data目录
    datadir=D:\mysql_home\data

    sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
    [WinMySQLAdmin]
    D:\mysql_home\bin\mysqld.exe
  ```
  
3. 添加环境变量
	MYSQL_HOME -- D:\mysql_home
	path后追加;%MYSQL_HOME%\bin
	
4. 安装mysql服务
	进入D:\mysql_home\bin目录以管理员身份打开控制台
	mysqld --console
	mysqld --initialize
	mysqld install
	
5. 启动mysql服务
	net start mysql
	(停止服务: net stop mysql)
	
6. 修改root账户密码
	修改my.ini，在[mysqld]下添加一行skip-grant-tables来跳过密码验证
	重启后进入mysql，执行update mysql.user set authentication_string=password('{{mypassword}}}') where user='root' and Host = 'localhost';
	可能会修改mysql.user.password_expired字段为否，来取消密码过期或者flush privileges
	删除skip-grant-tables后重启mysql
	以新密码登录

