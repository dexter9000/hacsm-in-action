持久化层架构
=============

这里我采用的是PostgreSQL作为持久层的技术方案，结合pqpool搭建一套持久层的集群环境。下表是postgreSQL数据几种实现集群的方法特性比较

| 特性  | 共享磁盘故障切换 | 文件系统复制 | 事务日志传送 | 基于触发器的主备复制 | 基于语句的复制中间件 | 异步多主服务器复制 | 同步多主服务器复制|
| ----  |:--------------:|:-----------:|:-----------:|:------------------:|:------------------:|:-----------------:|:---------------:|
| 常见的实现 | NAS | DRBD | Streaming Repl. | Slony | pgpool-II | Bucardo |  |
| 通信方式 | 共享磁盘 | disk blocks | WAL | table rows | SQL | 表锁 | 表锁，行锁|
| 不需要特殊的硬件 | | • | • | • | • | • | • |
| 允许多个主服务器 | | | | | • | • | • |
| 主服务器没有额外开销 | | • | | • | | • | |
| 无需等待多个服务器 | • | | with sync off | • | | • | |
| 主机故障不会丢失数据 | • | • | with sync on | | • | | • |
| 待命接受只读查询 | | | with hot | • | • | • | • |
| 表粒度 | | | | • | | • | • |
| 不需要解决冲突 | • | • | • | • | | | • |

# 共享磁盘故障切换
共享磁盘失效切换通过仅保存一份数据库副本来避免花在同步上的开销。 这个方案让多台服务器共享使用一个单独的磁盘阵列。 如果主服务器失效，备份服务器将立即挂载该数据库， 就像是从一次崩溃中恢复一样。这个方案允许快速的失效切换并且不会丢失数据。

共享硬件的功能通常由网络存储设备提供， 也可以使用完全符合POSIX行为的网络文件系统(参阅第 17.2.1 节)。 这种方案的局限性在于如果共享的磁盘阵列损坏了， 那么整个系统将会瘫痪。 另一个局限是备份服务器在主服务器正常运行的时候不能访问共享的存储器。

# 文件系统复制（块设备）
一种改进的方案是文件系统复制：对文件系统的任何更改都将镜像到备份服务器上。 这个方案的唯一局限是必须确保备份服务器的镜像与主服务器完全一致— 特别是写入顺序必须完全相同。DRBD是Linux上的一种流行的文件系统复制方案。

# 事务日志传送
热备份服务器可以通过读取WAL记录流来保持数据库的当前状态。如果主服务器失效，那么热备份服务器将包含几乎所有主服务器的数据，并可以迅速的将自己切换为主服务器。这是一个异步方案，并且只能在整个数据库服务器上实施。

使用基于文件的日志传送或流复制，或两者相结合。

# 基于触发器的主备复制
这个方案将所有修改数据的请求发送到主服务器。 主服务器异步向从服务器发送数据的更改信息。 从服务器在主服务器运行的情况下只应答读请求。对于数据仓库的请求来说， 从服务器非常理想的。

Slony-I是这个方案的一个例子，它支持针对每个表的粒度并支持多个从服务器。 因为它异步、批量的更新从服务器， 在失效切换的时候可能会有数据丢失。

# 基于语句的复制中间件
可以使用一个基于语句的复制中间件程序截取每一个SQL查询， 并将其发送到某一个或者全部服务器。每一个服务器都独立运行。 读-写请求发送给所有服务器，所以每个服务器接收到任何变化。但是只 读请求则仅发送给某一个服务器，从而实现读取的负载均衡。

如果只是简单的广播修改数据的SQL语句， 那么类似random(), CURRENT_TIMESTAMP 以及序列函数在不同的服务器上将生成不同的结果。 这是因为每个服务器都独立运行并且广播的是SQL语句而不是如何对行进行修改。 如果这种结果是不可接受的，那么中间件或者应用程序必须保证始终从同 一个服务器读取这些值并将其应用到写入请求中。 另外还必须保证每一个事务必须在所有服务器上全部提交成功或者全部回滚， 或者使用两阶段提交(PREPARE TRANSACTION 和COMMIT PREPARED)。 Pgpool-II和Continuent Tungsten是这种方案的实例。

# 异步多主服务器复制
对于那些不规则连接的服务器(比如笔记本电脑或远程服务器)，要在它们之间保持数据一致是很麻烦的。 在这个方案中，每台服务器都独立工作并周期性的与其他服务器通信以识别相互冲突的事务。 可以通过用户或者冲突判决规则处理出现的冲突。

# 同步多主服务器复制
在这种方案中，每个服务器都可以接受写入请求，修改的数据将在事务被提交之前必须从原始服务器广播到所有其它服务器。 过多的写入动作将导致过多的锁定，从而导致性能低下。 事实上，在多台服务器上同时写的性能总是比在单独一台服务器上写的性能低。读请求将被均衡的分散到每台单独的服务器。 某些实现使用共享磁盘来减少通信开销。同步多主服务器复制方案最适合于读取远多于写入的场合。它的优势是每台服务器都能接受写请求—因此不需要在主从服务器之间划分工作负荷。因为在服务器之间发送的是数据的变化，所以不会对非确定性函数(比如random())造成不良影响。

PostgreSQL不提供这种类型的复制。 但是PostgreSQL的两阶段提交(PREPARE TRANSACTION和 COMMIT PREPARED) 可以用于在应用层或中间件代码中实现这个功能。

这里可以看出对于不同的实现场景需要采用不同的解决方案，没有

## 1.准备操作系统

### 1 安装操作系统
这里为了操作方便，安装的是ubuntu 16.04 service版（小版本号可以忽略），为了便于使用，事先安装了gcc和sshd。安装镜像的下载地址：[Ubuntu Server](http://www.ubuntu.org.cn/download/server)

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
### 1 安装PostgreSQL
  输入如下命令
```
 sudo apt-get install postgresql
```
系统会提示安装所需磁盘空间，输入"y"，安装程序会自动完成。 安装完毕后，系统会创建一个数据库超级用户“postgres”, 密码为空。这个用户既是不可登录的操作系统用户，也是数据库用户。

修改Linux用户postgres的密码
  ```
  sudo passwd postgres  ```

### 2 创建数据库实例
安装完成后会默认创建一个数据库实例，并默认自启动，这里我们自己重新创建三个实例

停用默认的数据库实例
```
sudo update-rc.d postgresql disable```

关闭数据库进程
```
sudo /etc/init.d/postgres stop```

切换到Linux下postgres用户
```
   sudo su postgres```

在根路径创建用于保存数据库实例的目录
```
sudo mkdir /data```

然后创建两个数据库实例
```
/usr/lib/postgresql/9.5/bin/pg_ctl init -D /data/pgdb1 ```

设置其它机器上对postgres的访问，修改``/data/pgdb1/pg_hba.conf``
```
host    all      all    192.168.0.0/16    trust
# 0.0.0.0为地址段，0为多少二进制位
# 例如：192.168.0.0/16代表192.168.0.1-192.168.255.254```

修改 ``/etc/postgresql/8.4/main/postgresql.conf``
```
listen_address = '*'    #监听地址
port = 5436             #监听端口```

修改数据库超级用户postgres的密码

登录postgres数据库
```  
  psql postgres```
出现postgres的命令行提示符：``postgres=#``
输入如下命令
```
 ALTER USER postgres with PASSWORD 'password';```
键入"\q"返回到Linux命令行。

在Linux环境执行以下命令添加新用户，并按照提示输入该用户的密码
```
 createuser -drSP demo```

创建一个属于自定义用户的数据库
```
 createdb -O demo demodb```
启动数据库实例
```
/usr/lib/postgresql/9.5/bin/pg_ctl start -D /data/pgdb1 > /data/pgdb1.log```

最后按照上面的步骤创建另外一个实例

### 3 安装pgpool-II
这里直接通过apt-get安装，官网上是通过源码安装的，但是安装过程中的配置比较麻烦，都用apt-get安装能够减少配置。
```
apt-get install postgresql-9.5-pgpool2```
配置``pcp.conf``
首先创建md5码
```
pg_md5 -u pgpool -p ```
把结果复制到pcp.conf文件里，例如
```
# USERID:MD5PASSWD
pgpool:ba777e4c2f15c11ea8ac3be7e0440aa0```

配置pgpool.conf
复制样例文件
```
cd /opt/pgpool/etc
cp pgpool.conf.sample pgpool.conf```

修改配置文件
主节点pgpool.conf
```
listen_addresses = '*'
port = 9999
backend_hostname0 = '192.168.222.150'   ##配置数据节点 db1
backend_port0 = 5432
backend_weight0 = 1
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = '192.168.222.150'   ##配置数据节点  db2
backend_port1 = 5433
backend_weight1 = 1
backend_flag1 = 'ALLOW_TO_FAILOVER'
delegate_IP = '192.168.2.200'   ## 配置 pgpool 的 VIP，避免 pgpool 的单点故障
```

我的pgpool安装的时候没有创建进程跟踪和日志文件的目录，这里需要手工创建一下，并设置postgres为owner
```
sudo mkdir /var/run/pgpool
sudo mkdir /var/log/pgpool
chown postgres /var/run/pgpool
chown postgres /var/log/pgpool
```

# 启动和停止pgpool

通过`pgpool`和`pgpool stop`命令就可以启动和停止pgpool服务


License
---------  
-  本文档遵守 GUN license. 详情请查看LICENSE.

