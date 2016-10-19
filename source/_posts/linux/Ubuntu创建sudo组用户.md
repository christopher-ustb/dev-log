---
title: Ubuntu创建sudo组用户
date: 2016-07-22 10:36:24
categories: 
- Linux
tags:
---

# Ubuntu创建sudo组用户

1. adduser username
	* Enter new Unix password:
	* Changing the user information for lebooks此处一路enter全部接受默认设置即可
	* Is the information correct?[Y/n]
2. usermod -aG sudo username 将用户加入sudo组，a：append，追加到新的组而不将其从原来的组移除；G：group
3. su - username 测试用户权限


## 番外篇：为什么使用sudo用户而不是直接使用root

1. 在日常使用中直接以root用户身份是十分危险的，一个错误的指令可能在不经意间摧毁系统。理想情况是，你以一个仅有你工作需要的权限的用户来操作Linux，当使用sudo时，Think before doing.

2. sudo会在/var/log/auth.log留下命令日志

3. 锁定root用户，让暴力破解root密码的攻击无处下手，因为一个系统肯定有root用户，但是sudo用户的用户名却无处可猜

4. 多用户允许更方便地改变admin权限

5. sudo能更细致地设置安全策略

6. 如果以sudo用户登录后离开电脑与以root身份登录的危害不同

7. 以root身份运行的网络应用如果被破解入侵并植入脚本，将造成难以挽回的损失（血的教训！）

[参考资料:Ubuntu help](https://help.ubuntu.com/community/RootSudo)
