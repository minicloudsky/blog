---
title: docker学习笔记
date: 2019-05-15 08:33:42
tags:
- docker
categories: docker
---

### 什么是docker
docker包括一个命令行程序、一个后台守护进程、以及一组远程服务。它解决了常见的软件问题，并建华路安装、运行、发布和删除软件。这一切能够实现是使用一项Unix技术，称为容器。
### 容器不是虚拟化
在没有docker时代，使用硬件虚拟化(虚拟机),以提供隔离，虚拟机提供虚拟硬件，可安装一个os和其他程序，需要很长时间创建，资源开销打，因为除了要执行你需要的软件，还需要运行整个操作系统的副本。
docker容器不使用硬件虚拟化，运行在docker容器中的程序接口和逐渐的Linux内核直接打交道，容器中运行的程序与计算机操作系统直接没有额外的中间层，没有资源被冗余软件的运行或者虚拟硬件模拟而浪费掉，docker不是虚拟化技术，相反，它可以帮助使用已经内置到操作系统中的容器技术。

<!-- more -->

### 隔离容器中运行软件
docker使用已经成为linux一部分的linux命名空间和cgroups，docker不提供容器技术，但它使得容器更易于使用要了解系统中在容器什么样子，让我们先建立一条基线，如下为计算机系统体系结构上运行的基本容器示例。
![](instance.png)

运行docker可以认为是在用户控件运行两个程序，首先是docker守护进程，如果正确安装，该进程应该始终处于运行状态。另一个为docker CLI，它是与用户交互的docker程序，如果要启动，停止，运行或安装软件，你可以用docker CLI执行相应命令。
![](three.png)

### docker容器隔离
docker构建容器隔离包括8个方面，具体如下:  

* `PID命令空间` ----- 进程标识符和能力
* `UTS命名空间` ----- 主机名和域名
* `MNT命名空间` ----- 文件系统访问和结构
* `IPC命名空间` ----- 通过共享内存的进程间通信
* `NET命名空间` ----- 网络访问和结构 
* `chroot()`   ----- 控制文件系统根目录的位置
* `cgroups`    ----- 资源保护  

Linux命名空间和`cgroups`管理着运行时容器，docker采用另一套技术，就像运输集装箱一样为文件提供容器。  
### 分发容器
docker容器可以看成物理运输的集装箱，这是你存储、运行应用程序及其所有依赖的盒子。docker可以执行、复制、轻松地分发容器。docker通过一种打包和分发软件，完成传统容器的封装。这个用来充当容器分发角色的组件被称为`镜像`。  
Docker镜像，是一个容器中运行程序的所有文件的捆绑快照。你可以从镜像中创造尽可能多的容器，但是你这样做时候，从相同的镜像启动的容器不共享文件系统的更改。当你用docker分发软件，其实就是分发这些镜像，并在接收的机器上创建容器镜像在docker生态系统中是可交付的基本单位。



------

### docker下安装常用工具命令总结

查看docker磁盘占用

`docker system df`

查看虚悬镜像

`docker image ls  -f dangling=true`

删除虚悬镜像

`docker image  prune`

根据仓库名列出镜像

`docker images ls Ubuntu`

列出镜像和标签

`docker images ls Ubuntu:18.04`

容器重命名 

docker rename name1 name2

## Dockerfile

from 指定基础镜像

以一个镜像为基础，在其上进行定制。必须是第一条命令

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为scratch。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

`FROM scratch`

如果你以scratch为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在

   ### CMD 容器启动命令

   `CMD` 指令的格式和 `RUN` 相似，也是两种格式：

   - `shell` 格式：`CMD <命令>`

- `exec` 格式：`CMD ["可执行文件", "参数1", "参数2"...]`
   - 参数列表格式：`CMD ["参数1", "参数2"...]`。在指定了 `ENTRYPOINT` 指令后，用 `CMD` 指定具体的参数。

   之前介绍容器的时候曾经说过，Docker 不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。`CMD` 指令就是用于指定默认的容器主进程的启动命令的。

   在运行时可以指定新的命令来替代镜像设置中的这个默认命令，比如，`ubuntu` 镜像默认的 `CMD` 是 `/bin/bash`，如果我们直接 `docker run -it ubuntu` 的话，会直接进入 `bash`。我们也可以在运行时指定运行别的命令，如 `docker run -it ubuntu cat /etc/os-release`。这就是用 `cat /etc/os-release` 命令替换了默认的 `/bin/bash` 命令了，输出了系统版本信息。

   在指令格式上，一般推荐使用 `exec` 格式，这类格式在解析时会被解析为 JSON 数组，因此一定要使用双引号 `"`，而不要使用单引号。

   ### ENTRYPOINT 入口点

   `ENTRYPOINT` 的格式和 `RUN` 指令格式一样，分为 `exec` 格式和 `shell` 格式。

   `ENTRYPOINT` 的目的和 `CMD` 一样，都是在指定容器启动程序及参数。`ENTRYPOINT` 在运行时也可以替代，不过比 `CMD` 要略显繁琐，需要通过 `docker run` 的参数 `--entrypoint` 来指定。

   当指定了 `ENTRYPOINT` 后，`CMD` 的含义就发生了改变，不再是直接的运行其命令，而是将 `CMD` 的内容作为参数传给 `ENTRYPOINT` 指令，换句话说实际执行时，将变为：
   
   ```bash
   <ENTRYPOINT> "<CMD>"
   ```
当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：
检查本地是否存在指定的镜像，不存在就从公有仓库下载
利用镜像创建并启动一个容器
分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
从地址池配置一个 ip 地址给容器
执行用户指定的应用程序
执行完毕后容器被终止


#### mysql

##### 拉取mysql镜像

`docker pull mysql`

启动一个镜像，并将mysql映射到服务器3306端口

```bash
docker  run  -it --name  mysql -p 3306:3306   -e MYSQL_ROOT_PASSWORD=root   -v   /home/containermaps/mysql/log/:/var/log/mysql  -v /home/containermaps/mysql/conf.d/:/etc/mysql     -v   /home/containermaps/mysql/data:/var/lib/mysql -d mysql:5.7
```

默认用户为root

--name为容器名

-p指定端口

MYSQL_ROOT_PASSWORD 指定密码

进入mysql

`docker exec -it mysql5.7-2  bash`

`mysql     -uroot  -p`

#### nginx服务器

`docker pull nginx`

运行nginx

`docker run --name  nginx-test -p 80:80 -d nginx`

将nginx映射到本地目录

首先在/usr/local下新建目录

```bash
mkdir -p /usr/local/docker-nginx/www /usr/local/docker-nginx/logs /usr/local/docker-nginx/conf

将docker中nginx配置文件复制到到服务器

docker cp 63185c51103a:/etc/nginx/nginx.conf /usr/local/docker-nginx/conf/

关闭上一个容器，重新启动一个nginx容器

docker run -d -p 80:80 --name nginx-server -v /usr/local/docker-nginx/www:/usr/share/nginx/html -v /usr/local/docker-nginx/conf/nginx.conf:/etc/nginx/nginx.conf -v /usr/local/docker-nginx/logs:/var/log/nginx nginx
```

由此配置好以后，修改/usr/local/docker-nginx目录下的conf中的nginx.conf，就可以完成对nginx的配置，修改www目录下的html文件即可更新nginx部署的网页

#### redis数据库

```bash
docker pull redis
docker run --restart=always -d -p 6379:6379 -v /usr/local/docker-redis/data/:/data --name redis redis redis-server --appendonly yes --requirepass "redis"
```



##### 命令说明：

`-p 6388:6379 : 将容器的 6379 端口映射到主机的 6379 端口`
`-v /usr/local/docker-redis/data/ : 将主机中 /usr/local/docker-redis/data/ 目录下的 redis 挂载到容器的 /data`
`redis-server --appendonly yes : 在容器执行 redis-server 启动命令，并打开 redis 持久化配置`

连接redis

`docker exec -ti 5f4c4cf5a5f5 redis-cli a "redis passwd"`

[https://luoliangdsga.github.io/2018/04/26/%E4%BD%BF%E7%94%A8Docker%E9%83%A8%E7%BD%B2Redis/](https://luoliangdsga.github.io/2018/04/26/使用Docker部署Redis/)

#### mongodb

`docker pull mongo`

```bash
docker run -p 27017:27017 -v /usr/local/docker-mongodb/data:/data/db -d mongo
```

进入mongo

`docker exec -it mongo mongo admin`

### elasticsearch

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.5.2
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.5.2
```

### kibana

```bash
docker pull docker.elastic.co/kibana/kibana:7.6.0

docker run --link YOUR_ELASTICSEARCH_CONTAINER_NAME_OR_ID:elasticsearch -p 5601:5601 {docker-repo}:{version}
```

### docker 容器中软件安装

- 使用国内镜像

```
mv /etc/apt/sources.list /etc/apt/sources.list.bak
echo "deb http://mirrors.163.com/debian/ jessie main non-free contrib" >> /etc/apt/sources.list
echo "deb http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list
echo "deb-src http://mirrors.163.com/debian/ jessie main non-free contrib" >>/etc/apt/sources.list
echo "deb-src http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list
```

- 更新 `apt-get` 指令

```
apt-get update
```

- 安装 `yum` 命令

```
apt-get install vim
```

- 添加 `ls` 命令

```
vim ~/.bashrc
```

- 在最后一行添加

```
alias ll='ls $LS_OPTIONS -l'
```

- 使之生效

```
source ~/.bashrc
```

### postgresql

```bash
docker run -p 80:80 \
    -e 'PGADMIN_DEFAULT_EMAIL=1397991131@qq.com' \
    -e 'PGADMIN_DEFAULT_PASSWORD=pgadmin' \
    -e 'PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION=True' \
    -e 'PGADMIN_CONFIG_LOGIN_BANNER="Authorised users only!"' \
    -e 'PGADMIN_CONFIG_CONSOLE_LOG_LEVEL=10' \
    -d dpage/pgadmin4
```

```bash
 docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=root -d -p 5432:5432 -v /home/containermaps/postgres/:/var/lib/postgresql/data  postgres
```

### orcale

```bash
docker pull alexeiled/docker-oracle-xe-11g
docker run -h "oracle" --name "oracle" -d -p 49160:22 -p 49161:1521 -p 49162:8080 alexeiled/docker-oracle-xe-11g
```

create user orcale identified by orcale;



