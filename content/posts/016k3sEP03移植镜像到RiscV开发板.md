+++
date = '2025-04-02T19:10:13+08:00'
title = 'K3sEP03——移植镜像到RiscV开发板'
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

![](/img/jb/coffee.webp)

### 融入k3s

由于k3s的容器运行时采用containerd这个轻量化的原生工具而不建议使用docker，所以我们需要针对k3s的`registries.yaml`文件进行修改,将镜像地址全部修改为我们上面搭建的私有仓库地址。

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://192.168.173.76:6000" # 这里后续我们要思考如何将静态的IP地址写成动态或者如何面对IP变化的情况
      - "docker.1ms.run" # 保险起见设置一个其他的镜像仓库
    rewrite:
      "^riscv64/(.*)": "riscv64/$1"
  "192.168.173.76:6000":
    endpoint:
      - "http://192.168.173.76:6000"
```

之后再使用crictl命令进行拉取时就可以直接`crictl pull <镜像名称>`,注意，由于我们的镜像前面都是标有仓库地址的，所以在`crictl pull`时直接使用镜像名称即可。

### 常用操作

检查私有仓库中有哪些镜像。

```bash
curl -X GET http://192.168.173.76:6000/v2/_catalog
# 结果如下
{"repositories":["busybox","mac_riscv64/alpine","riscv64/busybox"]}
```

## 多平台构建适配的镜像

首先我们先抛开k3s,使用docker来进行移植适配操作，毕竟二者没有太大的差异，而docker可以提供更强大的性能。操作流程来自于[Docker官方文档](https://docs.docker.com/build/building/multi-platform/#simple-multi-platform-build-using-emulation)中的使用QEMU进行多平台镜像构建方法，这是官方文档中三种方法之一，希望后面我们可以尝试另外两种方法。

这次我们要适配的镜像是redis和nginx，操作的主机是Macbook M2,使用Docker Desktop方便操作。

### 提前配置

根据文档中的说法，我们需要先从classic镜像模式转换为containerd镜像模式，这在Docker Desktop上很好操作。

### 操作流程

1.新建目录保存redis等中间件的源码，为其编写的Dockerfile，以及可能自定义的配置文件等文件。

```bash
mkdir -p multi-platform
```

2.为了防止容器内部下载速度过慢，提前下载好redis的源码文件并解压到本机。

```bash
wget https://download.redis.io/releases/redis-7.2.4.tar.gz
tar -xzvf redis-7.2.4.tar.gz
```

3.编写Dockerfile文件，这里要注意我们使用`alpine:latest`作为基础镜像，这个镜像是我尝试的几个基础镜像中适配riscv最好的；同时需要替换默认的apk包镜像源，否则会遇到`could not connect server`问题；为了设置密码等一些自定义配置，我们自行编写一个配置文件到当前目录中`redis.conf`

```conf
# 允许所有 IP 连接（不要绑定 127.0.0.1）
bind 0.0.0.0

# 关闭 protected mode，否则外部无法连接
protected-mode no

# 设置访问密码
requirepass password

# 设置持久化 RDB 保存策略
save 900 1
save 300 10
save 60 10000

# RDB 文件名
dbfilename dump.rdb

# RDB 文件保存路径
dir ./

# 日志级别
loglevel notice

# 日志输出到标准输出（容器中建议这样）
logfile ""

# 后台运行，容器中不能开启
daemonize no

# 最大连接数（可选）
maxclients 10000

# 设置内存上限（可选）
# maxmemory 256mb
# maxmemory-policy allkeys-lru

```

这是Dockerfile:

```dockerfile
FROM alpine:latest

# 替换默认 apk 源为镜像源
RUN sed -i 's#https\?://dl-cdn.alpinelinux.org/alpine#http://mirrors.aliyun.com/alpine#g' /etc/apk/repositories

# 安装 Redis 构建依赖
RUN apk update && apk add --no-cache build-base jemalloc-dev linux-headers

# 拷贝 Redis 源码（假设 redis-7.2.4 文件夹和 Dockerfile 同目录）
COPY redis-7.2.4 /redis
WORKDIR /redis

# 编译 Redis
RUN make

# 拷贝配置文件
COPY redis.conf /etc/redis.conf

# 设置默认启动命令
CMD ["src/redis-server", "/etc/redis.conf"]
```

4.构建跨平台镜像。这一步本质上用到了buildx和QEMU，但是Docker Desktop为我们把这些底层都封装了起来，所以如果你是用Docker Desktop启动的Docker就不用在意;构建的同时如果遇到网络问题，请在参数中添加`--network host`命令，这会将宿主机网络作为容器内部网络，绕过网络限制。这个解决思路来自于[stackoverflow](https://stackoverflow.com/questions/53710206/docker-cant-build-because-of-alpine-error)

```bash
# 这里我们直接-t打好了标签，不用再tag了
docker build --network host --platform=linux/riscv64 -f Dockerfile.redis -t 192.168.173.76:6000/riscv64/redis:1.1 .
```

5.推送到本地私有镜像仓库中，再从RiscV开发板中拉取该镜像,这一步和上面私有镜像仓库构建相同。

```bash
docker push 192.168.173.76:6000/riscv64/redis:1.1
# riscv下
docker pull 192.168.173.76:6000/riscv64/redis:1.1
```

6.运行redis-server容器，并在主机上运行redis-cli来测试

```bash
docker run -d --name redis-test -p 6379:6379 192.168.173.76:6000/riscv64/redis:1.1
# 主机运行
redis-cli -h <riscvIP> -p 6379
> AUTH password
> set foo bar
> get foo
```

得到最终结果"bar",证明我们成功了！

同理我们也可以测试一下nginx镜像，下面是其简单的dockerfile

```dockerfile
FROM alpine:latest

# 替换源（可选，也可以用清华源）
RUN sed -i 's#dl-cdn.alpinelinux.org#mirrors.aliyun.com#g' /etc/apk/repositories \
    && apk update && apk add --no-cache nginx

# 拷贝配置文件和静态资源
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/

# 暴露端口
EXPOSE 80

# 启动 Nginx
CMD ["nginx", "-g", "daemon off;"]
```

其中的nginx.conf

```bash
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
}
```

对应的html文件内容:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Welcome to RISC-V Nginx!</title>
    <style>
        body {
            font-family: "Segoe UI", sans-serif;
            background-color: #f4f4f4;
            text-align: center;
            padding-top: 100px;
        }
        h1 {
            color: #333;
        }
        p {
            color: #777;
        }
        .tag {
            display: inline-block;
            margin-top: 20px;
            padding: 8px 16px;
            background-color: #007acc;
            color: white;
            border-radius: 4px;
            font-weight: bold;
        }
    </style>
</head>
<body>
    <h1>🚀 Hello from RISC-V Nginx!</h1>
    <p>Your nginx server is up and running.</p>
    <div class="tag">riscv64 / alpine / nginx</div>
</body>
</html>
```

同理打包推送拉取。运行容器后，在我的mac主机上访问`http://<riscvIP>:8080`可以看到上述html文件的效果

![](/img/riscv/nginx01.png)

### 优化镜像

当前redis镜像大小为146MB

```bash
(base) lutao@MagicTech multi-platform % docker images | grep redis
192.168.173.76:6000/riscv64/redis     1.1       e8e1336c3ccf   13 hours ago    146MB
```

我们可以采用多阶段构建以精简最终镜像，使其不包含构建工具和源代码。

```dockerfile
# ---------- 构建阶段 ----------
FROM alpine:latest AS builder

# 使用阿里源加速
RUN sed -i 's#http.*/alpine#http://mirrors.aliyun.com/alpine#g' /etc/apk/repositories

# 安装构建依赖
RUN apk update && apk add --no-cache build-base jemalloc-dev linux-headers

# 拷贝源码
COPY redis-7.2.4 /redis
WORKDIR /redis

# 编译 Redis
RUN make

# ---------- 运行阶段 ----------
FROM alpine:latest

# 使用阿里源（可选）
RUN sed -i 's#http.*/alpine#http://mirrors.aliyun.com/alpine#g' /etc/apk/repositories

# 安装运行所需的 jemalloc
RUN apk add --no-cache jemalloc

# 拷贝 redis-server 和 redis-cli（如果你需要 CLI 工具）
COPY --from=builder /redis/src/redis-server /usr/local/bin/
COPY --from=builder /redis/src/redis-cli /usr/local/bin/

# 拷贝配置文件
COPY redis.conf /etc/redis.conf

# 设置默认启动命令
CMD ["redis-server", "/etc/redis.conf"]
```

最终构建过程花费293.4s

![](/img/riscv/redis01.png)

最终镜像大小只有10.2MB，相较于之前的146MB缩小了许多。

![](/img/riscv/redis02.png)

同时我们也可以构建一个只有redis-cli的镜像，对于融合进k3s集群中或许可以在server节点上使用redis-server镜像，然后在其他agent节点的pod中部署redis-cli对其进行访问。

## 总结

搭建了一个简易的私有镜像仓库和移植了redis-server镜像到RiscV机器上，算是迈出了不错的一步，后面的工作主要围绕着如何将其更深入地融入到k3s中以及深入探究docker的多平台构建其底层原理展开。

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**