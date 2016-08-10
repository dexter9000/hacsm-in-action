持久化层架构
=============

这里我采用的是PostgreSQL作为持久层的技术方案，结合pqpool搭建一套持久层的集群环境。

## 1.准备操作系统

### 1 安装操作系统
这里为了操作方便，安装的是ubuntu 16.04 service版（小版本号可以忽略），为了便于使用，事先安装了gcc和sshd。

### 2 为网络节点准备环境
- 首先需要为每一个网络节点设置hostname以便区分。
```
sudo vim /etc/hostname
```


- 为每一个网络节点设置不同的ip地址
```
sudo vim /etc/network/interfaces
```

- 增加如下内容
```
iface enp0s3 inet static
address 192.168.222.150
netmask 255.255.255.0
gateway 192.168.222.255
```

- 重启机器

## 3 安装postgresql服务
### 1. 安装PostgreSQL
  输入如下命令
```
 sudo apt-get install postgresql
```
系统会提示安装所需磁盘空间，输入"y"，安装程序会自动完成。 安装完毕后，系统会创建一个数据库超级用户“postgres”, 密码为空。这个用户既是不可登录的操作系统用户，也是数据库用户。
### 2. 修改Linux用户postgres的密码
  输入如下命令
  ```
  sudo passwd postgres
  ```
###  3. 修改数据库超级用户postgres的密码
  1) 切换到Linux下postgres用户
   sudo su postgres
  2) 登录postgres数据库
  psql postgres


  License
---------  
  本文档遵守 GUN license. 详情请查看LICENSE.
