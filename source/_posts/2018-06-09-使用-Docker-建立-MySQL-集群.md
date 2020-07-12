---
title: 使用 Docker 建立 MySQL 集群
date: 2018-06-09 06:33:58
categories:
- '应用实践'
tags:
- 'Linux'
- 'Docker'
- 'MySql'
---
**官方说明是实验性质，生产环境应用还请自行斟酌**

没事干，突然想到 MySQL 集群，上网搜了一下看看想搭一个玩一下。

不过人在公司，刚好昨晚 autossh 还没弄好。。所以就直接用了最偷懒的方式搭建了。这里用傻瓜化的步骤来记录一下我搭集群的步骤，这里贴出[官方的做法](https://hub.docker.com/r/mysql/mysql-cluster/)结合理解吧。

首先说一下结构，mysql 集群主要由三个部分协作形成，分别是一个 manager 节点，多个子节点，以及一个用于访问的门面节点，manager 节点对应 `ndb_mgmd` ，子节点对应 `ndbd`，门面节点就是常规的 `mysqld` 。

我建议首先配置好集群的配置文件，配置文件主要有两个

````
cattenlinger@cattenlinger-Office:/opt/mysql_cluster_docker$ ls -al
total 16
drwxr-xr-x 2 root root 4096 Apr 26 11:39 .
drwxr-xr-x 3 root root 4096 Apr 26 10:51 ..
-rw-r--r-- 1 root root  838 Apr 26 11:38 my.cnf
-rw-r--r-- 1 root root 1146 Apr 26 11:38 mysql-cluster.cnf
````

`my.cnf` 给子节点以及 mysql 门面用。主要内容是使用集群模式并设定 manager 节点 IP 

```Ini
[mysqld]
ndbcluster
ndb-connectstring=172.18.1.100
user=mysql

[mysql_cluster]
ndb-connectstring=172.18.1.100	
```

`mysql-cluster.cnf` 给 manager 节点和子节点用的，主要内容是配置节点数量，使用内存大小以及各个子节点的 IP 配置，包括 mysqld。

```ini
[ndbd default]
NoOfReplicas=4
DataMemory=128M
IndexMemory=32M


[ndb_mgmd]
NodeId=1
hostname=172.18.1.100
datadir=/var/lib/mysql

[ndbd]
NodeId=2
hostname=172.18.1.101
datadir=/var/lib/mysql

[ndbd]
NodeId=3
hostname=172.18.1.102
datadir=/var/lib/mysql

[ndbd]
NodeId=4
hostname=172.18.1.103
datadir=/var/lib/mysql

[ndbd]
NodeId=5
hostname=172.18.1.104
datadir=/var/lib/mysql

[mysqld]
NodeId=6
hostname=172.18.1.199
```

准备好这两个文件之后，接下来操作 docker，里面的文件路径请根据自己的实际情况作修改。

官方建议是创建一个私有子网络给集群内部使用，然后再开始创建节点，我偷懒我就写个 bash 算了 XD

```bash
#!/bin/bash

# 创建私有网络，上面提到的
docker network create mysql-cluster --subnet=172.18.1.0/24;

# 创建管理节点
docker create \
 --net=mysql-cluster \
 --name=mysql-mngr-0 \
 --ip=172.18.1.100 \
 -v /opt/mysql_cluster_docker/mysql-cluster.cnf:/etc/mysql-cluster.cnf \
 mysql/mysql-cluster ndb_mgmd;

# 创建 4 个子节点
for i in 1 2 3 4; do
    docker create \
     --net=mysql-cluster \
     --name=mysql-node-$i \
     --ip=172.18.1.10$i \
     -v /opt/mysql_cluster_docker/mysql-cluster.cnf:/etc/mysql-cluster.cnf \
     -v /opt/mysql_cluster_docker/my.cnf:/etc/my.cnf \
     mysql/mysql-cluster ndbd;
done;

# 创建门面节点
docker create \
 --net=mysql-cluster \
 --name=mysql-facade \
 --ip=172.18.1.199 \
 -p 3307:3306 \
 -v /opt/mysql_cluster_docker/my.cnf:/etc/my.cnf \
 -e MYSQL_RANDOM_ROOT_PASSWORD=true \
 mysql/mysql-cluster mysqld;
```

然后就可以开始启动了，启动是有顺序的，先启动管理节点，然后启动子节点，再启动门面节点。（继续是 bash。。。

```bash
#!/bin/bash

# 启动管理节点
docker start mysql-mngr-0;

# 启动各个子节点
for i in 1 2 3 4; do
  docker start mysql-node-$i;
done

# 启动门面节点
docker start mysql-facade;
```

大概等两分钟，你就会看到成功启动了，现在通过门面节点的 log 获取随机生成的 root 密码就好了。

```bash
docker logs mysql-facade 2>&1 | grep PASSWORD
```

然后登陆

```bash
docker exec -it mysql1 mysql -uroot -p
```

改密码、增加远程账户，然后就可以在外部链接到集群了。