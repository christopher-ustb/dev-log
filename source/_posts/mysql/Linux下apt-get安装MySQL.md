---
title: Linux下apt-get安装MySQL
date: 2016-03-03 11:36:24
categories: 
- Linux
tags:
- linux
- mysql
---

# Linux下apt-get安装MySQL

1. apt-get install mysql-server

2. apt-get install mysql-client

3. mysql -u root -p进入mysql命令行，用set password命令修改密码
	```sql
        SET PASSWORD FOR 'root'@'localhost'=PASSWORD(‘localhost login password');
        SET PASSWORD FOR'root'@'%'=PASSWORD(‘remote login password');
    ```

4. 大坑-mysql不能远程访问

	gedit /etc/mysql.my.cnf
	找到bind-address      =127.0.0.1
	修改为bind-address   =0.0.0.0

		```
			# Instead of skip-networking the default is now to listen only on
			# localhost which is more compatible and is not less secure.
			bind-address        = 0.0.0.0
		```
	授权远程ip登录
		```sql
			GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'%' IDENTIFIED BY 'mypassword' WITH GRANT OPTION;
			FLUSH   PRIVILEGES;
		```

5. 修改字符集解决中文乱码问题

	my.cnf中mysqld添加配置
		```
			character-set-server=utf8mb4
			init-connect = "SET NAMES utf8mb4"
		```