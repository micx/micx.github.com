---
layout: post
title: 多git账号配置
category: linux
published: true
---


##1. 创建rsa

用 ssh-keygen 命令生成一组新rsa

```bash
ssh-keygen -t rsa -C 'second@mail.com' -f id_rsa_second 
```

##2. 配置~/.ssh/config文件


```bash
#Default Git
Host defaultgit
HostName IP Address #域名也可
User think
IdentityFile ~/.ssh/id_rsa
 
#Second Git
Host secondgit
HostName IP Address #域名也可
User think
IdentityFile ~/.ssh/id_rsa_second
```

Host就是每个SSH连接的单独代号，IdentityFile告诉SSH连接去读取哪个私钥。

##3. 执行ssh-agent让ssh识别新的私钥。

```bash
ssh-add ~/.ssh/id_rsa_new
```

该命令如果报错：Could not open a connection to your authentication agent.无法连接到ssh agent，可执行ssh-agent bash命令后再执行ssh-add命令。
以后，在clone或者add remote的时候，需要把config文件中的host代替git@remoteaddress中的remoteaddress。

同时，你可以通过在特定的repo下执行下面的命令，生成区别于全局设置的user.name和user.email。

```bash
git config user.name "newname"
git config user.email "newemail" 
#git config --global --unset user.name 取消全局设置
#git config --global --unset user.email 取消全局设置
```

##4. 例子

#在同一机器不同目录下克隆远程同一个repo

```bash
cd /home/user1
git clone git@defaultgit:xxx.git
```

```bash
cd /home/user2
git clone git@secondgit:xxx.git
```

上面的两条clone命令，虽然关联到同一个repo，却是通过不同ssh连接，当然也是不同的git账号。

