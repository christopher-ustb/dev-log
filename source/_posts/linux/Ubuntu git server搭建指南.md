---
title: Ubuntu git server搭建指南
date: 2015-12-19 19:16:24
categories: 
- Linux
tags:
- git
- Ubuntu
- vcs
---

# Ubuntu git server搭建指南

1. Ubuntu安装git
	apt-get update	更新apt-get
	apt-get install git自动安装git
	git --version 检测git安装成功

2. 新建一个运行git的Linux用户
	adduser git 新建一个名为git的用户（设置密码）

3. 创建远端仓库
	cd /home/
	mkdir git
	cd git
	mkdir yourproject.git
	git init --bare --shared yourproject.git 初始化仓库
	注意，一定要保证git用户对yourproject.git下所有文件有读写权限（建议递归修改目录owner: chown git yourproject.git）

4. 本地安装git和tortoisegit

5. 设置本地通用git用户name email（可在tortoisegit > settings直接设置）

6. 本地产生SSH公钥与私钥

	在git-bash中运行: ssh-keygen -C "{{user.email}}" -t rsa
	(将{{user.email}}替换为上文中设置的git email)
	
	一路enter后，windows系统在users\{{loginUser.name}}\.ssh目录下可以看到生成的公钥id_rsa.pub与私钥id_rsa文件

7. 将公钥复制到Ubuntu上
	追加到/home/git/.ssh/authorized_keys文件尾部（第一次复制需要手动创建该文件）
	github用户可将公钥添加到[github key设置页面](https://github.com/settings/keys)
	
8. 本地使用SSH私钥推拉远端仓库
	运行PuTTYGen，在Conversions菜单中点击Import key，选择ssh-keygen生成的私钥文件所在位置，比如id_rsa文件
	点击Save private key按钮，将其保存为.ppk文件
	打开TortoiseGit > settings > git > remote，点击Add Key，选择前一步所保存的.ppk文件所在的位置即可
