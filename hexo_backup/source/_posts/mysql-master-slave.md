---
title: mysql 主从复制实践
date: 2020-09-05 16:41:41
tags:
- 数据结构
---


### mysql是常见的关系型数据库，为了方便把一个mysql数据源的数据复制到其他mysql host，复制特性就是为了解决这个问题的，slave mysql 服务器通过读取master 的binlog进行复制，写入slave mysql 服务器。
### [参考mysql官网 replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
1. 因为贫民窟少女只有两台服务器的原因，这里使用 docker 开启三个容器，一个master，两个slave，进行主从复制测试
2. 首先拉取 mysql 容器，这里 mysql 最新的版本为8.0.21
   ```bash
   [root@huawei ~]# docker pull mysql
   ```
   ![](1.png)
3. 在服务器上创建docker 文件目录与服务器本地文件目录的映射，稍后将mysql容器的文件映射到这个目录(这里是博主个人习惯喽，因为主要是为了防止容器丢失早晨数据丢失，把容器内的数据映射到服务器外面，这样就算服务器故障，下次把mysql容器重新挂载到这个mysql服务器上的目录就好了~)
5. 创建三个mysql目录
```bash
[root@huawei container]# pwd
/home/container
[root@huawei container]# ll
total 12
drwxr-xr-x 6 root root 4096 Sep  4 23:47 mysql_master
drwxr-xr-x 6 root root 4096 Sep  5 16:35 mysql_slave
drwxr-xr-x 6 root root 4096 Sep  5 16:27 mysql_slave2
```
6.  在每个 mysql 目录中创建 data、conf、log、mysql-files 三个目录
   <!-- more -->
```bash
[root@huawei container]# tree -L 2
.
├── mysql_master
│   ├── conf
│   ├── data
│   ├── log
│   └── mysql-files
├── mysql_slave
│   ├── conf
│   ├── data
│   ├── log
│   └── mysql-files
└── mysql_slave2
    ├── conf
    ├── data
    ├── log
    └── mysql-files

15 directories, 0 files
```
1. 创建 master mysql
   ```bash
   [root@huawei container]# docker run -it  --name mysql_master -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -v /home/container/mysql_master/mysql-files:/var/lib/mysql-files -v /home/container/mysql_master/conf:/etc/mysql -v /home/container/mysql_master/log:/var/log/mysql  -d mysql:latest
   ```
   #### 这里简单解释下 docker 命令，下面不再赘述
   - -i: 交互式操作。
   - -t: 终端。
   - --name 指定容器名称
   - -p 指定容器的端口映射，格式是: 宿主机 ip : 容器内 ip ,这里将 容器内的mysql 3306 端口映射到主机上的 3306端口
   - -v 指定容器的目录挂载，格式是: 宿主机目录: 容器目录，比如 `-v /home/container/mysql_master/mysql-files:/var/lib/mysql-files` 是将 服务器上的 `/home/container/mysql_master/mysql-files`目录挂载到容器内的`/var/lib/mysql-files` 目录
   - -d 指定容器服务在后台运行(注：加了 -d 参数默认不会进入容器，想要进入容器需要使用指令 docker exec)
2. 进入 master mysql 容器
```bash
[root@huawei container]# docker exec -it mysql_master /bin/bash
root@d9120d223e94:/# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.0.21 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```
这里我们可以看到master mysql 已经成功运行了，下面我们修改下 mysql 的配置文件，我的 mysql 配置文件在 `/etc/mysql` 目录下，由于上面我们进行了目录挂载，这里我们可以直接在宿主机上面修改  `my.cnf` ,在 my.cnf 添加以下配置:
```bash
[mysqld]
## 设置server_id，一般设置为IP，同一局域网内注意要唯一
server_id=100
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，可以随便取，最好有含义（关键就是这里了）
log-bin=edu-mysql-bin
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
## 从主机跳过错误的次数
slave_skip_errors=1062
## 注意！注意！注意！这里很重要! 下面进行解释
default_authentication_plugin=mysql_native_password
## mysql 最大连接数
max_connections=10000
## mysql 最大连接错误次数
max_connect_errors=10000
```
### 由于 mysql8 更新了安全策略，默认是采用`caching_sha2_password plugin` 进行加密传输的，这里我们主要是测试 master-slave replication,为了简单起见，我在上面配置文件里面加了`default_authentication_plugin=mysql_native_password` 从而设置默认的认证插件是mysql原生密码，关于mysql 安全策略，详细可以参考:
- [ Caching SHA-2 Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html)
- [ Setting Up Replication to Use Encrypted Connections](https://dev.mysql.com/doc/refman/8.0/en/replication-solutions-encrypted-connections.html)
8. 下面我们创建下两个 slave mysql 实例
  
   - 创建 slave mysql，slave mysql 端口为 `3307`
   ```bash
   [root@huawei mysql_master]# docker run -it  --name mysql_slave -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root -v /home/container/mysql_slave/mysql-files:/var/lib/mysql-files -v /home/container/mysql_slave/conf:/etc/mysql -v /home/container/mysql_slave/log:/var/log/mysql   mysql:latest
   ```

   - 创建 slave2 mysql,slave2 mysql 端口为 `3308`
    ```bash
        [root@huawei mysql_master]# docker run -it  --name mysql_slave2  -p 3308:3306 -e MYSQL_ROOT_PASSWORD=root -v /home/container/mysql_slave2/mysql-files:/var/lib/mysql-files -v /home/container/mysql_slave2/conf:/etc/mysql -v /home/container/mysql_slave2/log:/var/log/mysql   mysql:latest
    ```
9. 配置 slave mysql
    同样的，我们修改下容器 `/etc/mysql` 下面的 `my.cnf` 配置文件，因为我这里做了目录挂载，所以直接修改 `/home/container/mysql_slave/conf/my.cnf` 和 `/home/container/mysql_slave2/conf/my.cnf` 就好了，在两个slave mysql的 `my.cnf` 加入下面的配置文件
```bash
[mysqld]
## 设置server_id，一般设置为IP,注意要唯一
server_id=101
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=edu-mysql-slave1-bin
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
## relay_log配置中继日志
relay_log=edu-mysql-relay-bin
## log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates=1
## 防止改变数据(除了特殊的线程)
read_only=1
default_authentication_plugin=mysql_native_password
max_connections=10000
max_connect_errors=10000
```
`my.cnf` 配置文件修改好以后，重启这三个 mysql 容器
```bash
[root@huawei conf]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
f1960bbc70c8        0d64f46acfd1        "docker-entrypoint.s…"   About an hour ago   Up About an hour    33060/tcp, 0.0.0.0:3308->3306/tcp   mysql_slave2
3be82b352a15        0d64f46acfd1        "docker-entrypoint.s…"   18 hours ago        Up About an hour    33060/tcp, 0.0.0.0:3307->3306/tcp   mysql_slave
d9120d223e94        0d64f46acfd1        "docker-entrypoint.s…"   18 hours ago        Up About an hour    0.0.0.0:3306->3306/tcp, 33060/tcp   mysql_master
```
```bash
[root@huawei conf]# docker restart f1960bbc70c8 3be82b352a15 d9120d223e94
f1960bbc70c8
3be82b352a15
d9120d223e94
[root@huawei conf]# 
```
10. 开启 `master slave replication`
    首先我们查看下这三个 mysql 的 host ip
```bash
[root@huawei conf]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
f1960bbc70c8        0d64f46acfd1        "docker-entrypoint.s…"   About an hour ago   Up 2 minutes        33060/tcp, 0.0.0.0:3308->3306/tcp   mysql_slave2
3be82b352a15        0d64f46acfd1        "docker-entrypoint.s…"   18 hours ago        Up 2 minutes        33060/tcp, 0.0.0.0:3307->3306/tcp   mysql_slave
d9120d223e94        0d64f46acfd1        "docker-entrypoint.s…"   18 hours ago        Up 2 minutes        0.0.0.0:3306->3306/tcp, 33060/tcp   mysql_master
[root@huawei conf]# docker inspect --format='{{.NetworkSettings.IPAddress}}' f1960bbc70c8
172.17.0.4
[root@huawei conf]# docker inspect --format='{{.NetworkSettings.IPAddress}}' 3be82b352a15
172.17.0.3
[root@huawei conf]# docker inspect --format='{{.NetworkSettings.IPAddress}}' d9120d223e94
172.17.0.2
```
可以看到，master,slave,slave2 这三个 mysql的 ip分别是
```bash
master -> 172.17.0.2
slave  -> 172.17.0.3
slave2 -> 172.17.0.4
```
下面我们为 slave slave2 的 mysql 在master中添加用户，并授予 replication 权限
执行以下sql
```sql

CREATE USER 'slave' @'%' IDENTIFIED WITH mysql_native_password BY 'root';
GRANT REPLICATION SLAVE,
REPLICATION CLIENT ON *.* TO 'slave' @'%';
CREATE USER 'slave2' @'%' IDENTIFIED WITH mysql_native_password BY 'root';
GRANT REPLICATION SLAVE,
REPLICATION CLIENT ON *.* TO 'slave2' @'%';
```
可以看到执行成功，这里我们通过原生密码的方式创建 slave,slave2 两个用户，并且授予这两个用户 `REPLICATION SLAVE,REPLICATION CLIENT`权限。
![](2.png)
我们查看下 master mysql的状态
```sql
show master status;
```
![](5.png)
这里，记录下 File 和 Position 字段的值，后来会用到，
然后我们在 slave和salve2 mysql上面配置 master 服务器，这里的 `master_log_pos` 就是我们上面看到的master mysql 的日志位置，Position=156
在 slave mysql 上面执行
```sql
CHANGE MASTER TO master_host = '172.17.0.2',
master_user = 'slave',
master_password = 'root',
master_port = 3306,
master_log_file = 'edu-mysql-bin.000001',
master_log_pos = 156,
master_connect_retry = 30;
```
从而修改 slave mysql 的 master mysql，这里使用的是之前我们查看过的 容器的 host ip
![](3.png)
同样的，我们在 slave2 mysql 上面执行
```sql
CHANGE MASTER TO master_host = '172.17.0.2',
master_user = 'slave2',
master_password = 'root',
master_port = 3306,
master_log_file = 'edu-mysql-bin.000001',
master_log_pos = 156,
master_connect_retry = 30;
```
![](4.png)
可以看到已经执行成功，然后重启下这三个mysql
```bash
[root@huawei conf]# docker restart f1960bbc70c8 3be82b352a15 d9120d223e94
f1960bbc70c8
3be82b352a15
d9120d223e94
```
我们查看下master,slave,slave2 三台服务器的状态
```bash
show master status;
```

```bash
show slave status;
```

```bash
show slave status;
```
![](6.png)
![](7.png)
![](8.png)
我们开启下 slave，从而让 slave mysql 从 master 进行复制，在两台 slave mysql上面执行 
```sql
start slave;
```
![](9.png)
![](10.png)
如果出现SlaveIORunning 是 no 或者SlaveSqlRuuning是no的情况，可以查看下 master 的 file和position，然后更新下`stop slave;`然后`reset slave;`,先停止slave mysql 然后重置下，slave slave2 的master position和file
```sql
CHANGE MASTER TO master_host = '172.17.0.2',
master_user = 'slave',
master_password = 'root',
master_port = 3306,
master_log_file = 'edu-mysql-bin.000014',
master_log_pos = 11725,
master_connect_retry = 30;
```
就好了
![](11.png)
![](12.png)
![](13.png)
然后我们在master mysql上面新建个数据库`test_master_slave`,然后刷新slave 和 slave2，可以看到 mysql slave 的 replication 已经开始运行了~
![](14.png)
可以看到master 上面我们刚刚新建的 slave 库已经通过replication 复制到了两个slave库上面，大功告成~
![](15.png)
![](16.png)
## bug fix
如果出现Slave_SQL_Running：no的情况，可以执行从而跳过复制错误就好了。
```sql
stop slave; SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1; 
START SLAVE; 
show slave status;
```
