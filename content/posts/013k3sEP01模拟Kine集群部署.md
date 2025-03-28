+++
date = '2025-03-26T18:51:27+08:00'
title = 'K3sEP01模拟Kine集群部署'
categories = ["核心技术"]
tags = ["RiscV","实验"]
+++

本篇文章用来记录自己在虚拟机上搭建k3s集群的整个过程，并且使用了kine这个k3s专属的数据存储方法（代替了k8s中的etcd存储方式）

## 实验环境

本机: macos M2
虚拟机: 采用[Multipass](https://canonical.com/multipass)这个虚拟机软件，专门为Ubuntu而制作的虚拟化软件，可以很轻松地在macos上使用。

- 一台虚拟机Agent01作为数据库的后端服务器 1G/8G
- 一台虚拟机Server作为Server 2G/8G
- 一台虚拟机Agent02作为Agent 1G/4G

操作系统: Ubuntu22.04

由于是实验环境，我们全部的操作直接在root权限下进行，如果接下来某些操作遇到权限问题请添加 sudo。

那让我们直接摇滚进来吧！

![](/img/jb/coffee.webp)

## 部署

以下步骤来自于AI的指导并成功部署运行。

### MYSQL节点

由于k3s的轻量化要求，使用了kine作为存储方案，旨在可以不使用etcd进行集群管理，可以外置其他数据库，例如,sqlLite,mysql,postgresql等。

数据库里面存放了原本etcd会存放的信息，例如各个节点的信息。

```bash
apt update
apt install mysql-server -y

mysql -u root -p
```

配置数据库

```sql
create database k3s;
create user 'k3s' @ '%' identified by 'yourpassword'; # 替换为自己设置的密码
grant all privileges on k3s.* to 'k3s@'%'';
flush privileges;
```

修改Mysql配置文件允许远程连接

```bash
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl restart mysql
```

最后，使用`hostname -I`记录一下这台虚拟机的IP地址，方便后续连接操作。

### 使用kine部署k3s server

使用`--datastore-endpoint`参数指定外接数据库的地址。

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="mysql://k3s:yourpassword@tcp(yourMysqlIP:3306)/k3s" # 替换为自己的密码和上一步的IP地址
```

ps:这个方法可能会遇到网络问题，我们还可以采用离线的方式安装k3s，详情请见[另一篇文章]()

检查Server的状态

```bash
systemctl status k3s

kubectl get nodes
```

同时可以回到数据库节点检查表信息。

```sql
use k3s;
show tables;
select * from kine limit5 /G;
```

![](/img/riscv/kine01.png)

可以发现有kine这个表。

### 部署Agent节点

首先从Server中获取Token，`cat /var/lib/rancher/k3s/server/node-token`

将agent加入集群中由Server管理

```bash
curl -sfL https://get.k3s.io | K3S_URL="https://yourServ erIP:6443" K3S_TOKEN="K10eb92a0a28e01cdcb43aebadf54e4c647274509bb903266f8b5753acce221234d::server:ecda1824570feac881c9b27b98584c76" sh -
```

### 验证集群功能

在Server上执行

```bash
kubectl get nodes -o wide
```

![](/img/riscv/kine02.png)

可以看到Server和Agent节点都存在。

到这里就初步部署成功。

### 添加镜像代理

如果我们使用registry模式，即从docker.io,dockerhub上拉取镜像， 我们需要**同时在Server和Agent上添加**  `/etc/rancher/k3s/registries.yaml`文件

在其中添加如下配置，以使代理的方式拉取镜像。其中镜像地址填入 endpoint 下方，目前[可用的镜像地址](https://github.com/dongyubin/DockerHub)

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.1ms.run"
```

## 可能出现的问题

在运行过程中如果你的本机IP地址变化，可能会出现数据库Mysql的IP封禁问题

使用下面命令查看问题所在

```bash
sudo journalctl -u k3s --no-pager --lines=50
```

![mysql问题](/img/riscv/kine03.png)

可以发现`Error 1129: Host '192.168.64.2' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'"`，由于太多次连接Mysql失败触发Mysql的安全机制，封禁了k3s Server的IP地址。

所以我们需要从数据库层面解禁。

```sql
mysql> FLUSH HOSTS;
Query OK, 0 rows affected, 1 warning (0.03 sec)

mysql> SELECT host FROM performance_schema.host_cache WHERE host='192.168.64.2';
Empty set (0.06 sec)
```

之后便可以正常运行了。

## 总结

后续文章准备探索离线安装k3s以及在open-Rv这个组合上一系列的k3s适配工作流程展示。

![](/img/ys/捂嘴憋笑.webp)