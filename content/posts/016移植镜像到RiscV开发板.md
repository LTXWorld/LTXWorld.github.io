+++
date = '2025-04-02T19:10:13+08:00'
title = 'K3sEP03移植镜像到RiscV开发板'
categories = ["硬核技术"]
tags = ["Docker","Macos","RiscV"]
+++

## 引子

了解K3s的都知道，Pod，容器，镜像在K3s中的重要作用，所以我们首先要克服的难点正是如何让镜像们适配RiscV架构。通常，我们拉取镜像都来自于dockerhub,里面搜索确实有一些RiscV镜像，例如`riscv64/nginx,riscv64/redis`等，但是经检查，其实它并不是官方认证的镜像，也几乎没有被维护或者使用，所以我们需要自己去进行镜像的适配。
拿nginx镜像举例:

`docker search riscv64/nginx`

检查是否有可用的镜像

```bash
riscv64/nginx                            Official build of Nginx.                         0
nginx/nginx-ingress                      NGINX and  NGINX Plus Ingress Controllers fo…   102
nginx/nginx-prometheus-exporter          NGINX Prometheus Exporter for NGINX and NGIN…   49
nginx/nginx-ingress-operator             NGINX Ingress Operator for NGINX and NGINX P…   2
nginx/unit                               This repository is retired, use the Docker o…   65
nginx/nginx-quic-qns                     NGINX QUIC interop                               1
nginxproxy/nginx-proxy                   Automated nginx proxy for Docker containers …   161
nginx                                    Official build of Nginx.                         20719     [OK]
```

可以发现`riscv64/nginx`的Stars数量为0，也不是官方镜像，所以拉取失败。（即使使用代理地址，开发板上也会遇到网络问题，因为代理地址中也没保存非官方的镜像）

由此可见，自行构建合适的镜像迫在眉睫。

## 私有镜像仓库

Docker为用户提供了一种构建私有镜像仓库的能力，利用其registry镜像，我们可以搭建一个私人的镜像仓库，由此我们可以不再依赖于远程拉取镜像，在搭建K3s集群时，可以选择一个节点作为数据存储节点，既存储各个Agent与Server的数据，也存储我们需要的专属镜像数据。

由于registry这个镜像暂时也不适配RiscV架构（会出现经典的manifest问题）

所以我们先在arm(macos)上尝试一下私有镜像如何搭建。

### 私有镜像搭建实验

本次实验将Mac主机私有镜像库的物理机，一台RiscV开发板作为拉取镜像的机器。

**首先，在Mac上构建Registry容器。**

```bash
# 这里我选择6000端口作为本机端口（5000被系统占用，无法彻底杀死）
docker run -d -p 6000:5000 --restart=always --name registry registry:2
```

构建成功后，如果你使用DockerDesktop可以在其中容器部分发现容器已经在正常运行了。我们再进行一下相关测试`curl http://<macIP>:6000/v2/`,其会返回`"repository[]"`信息，证明成功。

这时我们随便拉下来一个镜像进行测试`docker pull busybox:latest`
对这个镜像进行打标签，并推送到私有仓库中。(打标签操作本质上是复制一份镜像并为其改名)

```bash
# 注意，将后续的macIP改为自己本机的IP地址
# 原本tag命令不需要IP，但是如果指向私有地址，则需要带上私有地址IP
docker tag busybox:latest <macIP>:6000/riscv64/busybox:1
# 将此镜像推送到私有仓库中
docker push <macIP>:6000/riscv64/busybox:1
```

结果如下

```bash
The push refers to repository [192.168.173.76:6000/riscv64/busybox]
be632cf9bbb6: Pushed
1: digest: sha256:c109a60479ed80d63b17808a6f993228b6ace6255064160ea82adfa01c36deba size: 527
```

同理可以使用上面的curl命令测试当前私有仓库中的镜像列表。

**其次，本次实验为了方便暂时使用HTTP，没有使用HTTPS,即带有TLS的方式。**并且本次实验的主机与开发板连在同一个子网（热点）下，后续我们会探究使用https的方式。

所以，我们需要在本机与开发板上的docker的配置文件中同时添加

```bash
{
  "insecure-registries": ["<macIP>:6000"]
}
```

然后，重启Docker`sudo systemctl restart docker`

最后，来到RiscV开发板上拉取镜像`docker pull <macIP>:6000/riscv64/busybox:1`，使用`docker images`进行检查发现其存在。

这样我们就做到了从私有仓库拉取了一个我们在开发板上拉取不到的镜像到开发板上，这对于我们日后的集群搭建工作起到了关键的作用。

### 常用操作

检查私有仓库中有哪些镜像。

```bash
curl -X GET http://192.168.173.76:6000/v2/_catalog
# 结果如下
{"repositories":["busybox","mac_riscv64/alpine","riscv64/busybox"]}
```