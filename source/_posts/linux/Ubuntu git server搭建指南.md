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
	* apt-get update	更新apt-get
	* apt-get install git自动安装git
	* git --version 检测git安装成功

2. 新建一个运行git的Linux用户
	* adduser git 新建一个名为git的用户（设置密码）
	* 改变git用户的shell登录行为
	    * vim /etc/passwd
	    * 将git用户记录如：
	    ```bash
	    git:x:1001:1001:,,,:/home/git:/bin/bash
	    ```
	    改成
	    ```bash
	    git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell
	    ```

3. 创建远端仓库
	* cd /home/
	* mkdir git
	* cd git
	* mkdir yourproject.git
	* git init --bare --shared yourproject.git 初始化仓库
	* 注意，一定要保证git用户对yourproject.git下所有文件有读写权限（建议递归修改目录owner: chown -R git yourproject.git）

4. 本地安装git和tortoisegit

5. 设置本地通用git用户name email（可在tortoisegit > settings直接设置）

6. 本地产生SSH公钥与私钥

	* 在git-bash中运行: ssh-keygen -C "{% raw %}{{user.email}}{% endraw %}" -t rsa(将{% raw %}{{user.email}}{% endraw %}替换为上文中设置的git email)
	* 一路enter后，windows系统在users\{{loginUser.name}}\.ssh目录下可以看到生成的公钥id_rsa.pub与私钥id_rsa文件

7. 将公钥复制到Ubuntu上

	* 追加到/home/git/.ssh/authorized_keys文件尾部（第一次复制需要手动创建该文件）
	* github用户可将公钥添加到[github key设置页面](https://github.com/settings/keys)
	
8. 本地使用SSH私钥推拉远端仓库
	* 运行PuTTYGen，在Conversions菜单中点击Import key，选择ssh-keygen生成的私钥文件所在位置，比如id_rsa文件
	* 点击Save private key按钮，将其保存为.ppk文件
	* 打开TortoiseGit > settings > git > remote，点击Add Key，选择前一步所保存的.ppk文件所在的位置即可

---
   
关于Linux /etc/passwd文件的说明：

```bash
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
libuuid:x:100:101::/var/lib/libuuid:
syslog:x:101:104::/home/syslog:/bin/false
messagebus:x:102:105::/var/run/dbus:/bin/false
ntp:x:103:109::/home/ntp:/bin/false
sshd:x:104:65534::/var/run/sshd:/usr/sbin/nologin
git:x:1000:1000:,,,:/home/git:/usr/bin/git-shell
mysql:x:105:113:MySQL Server,,,:/nonexistent:/bin/false
ftp:x:106:114:ftp daemon,,,:/srv/ftp:/bin/false
tomcat7:x:107:115::/usr/share/tomcat7:/bin/false
redis:x:108:116:redis server,,,:/var/lib/redis:/bin/false
```

这个文件记录了linux中每个用户的基本属性，每个用户对应一行文本。每行文本又被冒号分隔为7部分，分别是：
1. 用户名
2. 密码：一般会加密保存在/etc/shadow文件中，此处只显示一个占位符*或者x
3. 用户id
4. 组id
5. 用户描述
6. 主目录
7. 用户登录后执行的shell：如root用户通常是`/bin/bash`来指定bash作为shell解释器，一些伪用户用`/usr/sbin/nologin`来禁用登录，上文中git用户使用`/usr/bin/git-shell`来指定git-shell为解释器

---
