+++
date = '2024-11-18T18:49:09+08:00'
title = '玩转Docker系列——数据持久化'
categories = ["核心技术"]
tags = ["容器","docker"]
+++

# Docker_Storage

![](/img/ys/晚上好兄弟们.jpg)

大家好我是LTX，我猜在使用docker时你或多或少都会有疑问，我的数据被存到哪里去了？（什么?你不用docker?，滚出克😡）docker化之后为什么得到的镜像这么小，比我原始的程序要小这么多？特别是如果我将Mysql容器化，那它保存的记录都去哪里了？

今天我将进入Docker_Volumes的世界，与大家聊聊Docker是如何处理数据持久化的。

秉持着STFW&RTFM的精神，我从[官方的手册](https://docs.docker.com/engine/storage/)看起，并且辅助以测试实验来验证手册中的理论，设计实验部分使用了部分AI工具辅助。

本机环境:macos15.1，Docker version 27.2.0, build 3ab4256

## Storage

手册中一开始就说明了默认情况下，容器内部创建的所有文件都存储在可写容器层上(详见docker基础原理)，这意味着，**当该容器不存在时，数据不会持久化。**

那不会持久化可不行啊，我MYSQL关键点之一就是持久化啊，另一个容器要用我的数据结果你取不出来锁在容器里面了，这两个容器交互不就完犊子了吗？不行不行，得有个办法来实现数据的持久化，要不然我docker还怎么立足于容器生态江湖。

并且更蛋疼的是容器的可写层与运行容器的主机紧密耦合，难以轻易地将数据移动到其他地方。*这是因为可写层是容器运行时基于主机环境创建的一种临时存储结构，它的存在依赖于容器的运行状态，不像独立于容器的存储机制那样便于迁移数据。*

其实就是docker底层需要一个虚拟机运行嘛(见docker基础原理)，我们在主机上当然无法直接访问这个虚拟机，那我数据呢!?我想要让其他的人也用用啊！

![](/img/ys/你说什么.webp)

docker设计者也当然为我想到了这种情况，于是他提供了两种解决方法：

* 数据持久化到主机的磁盘上
* 数据持久化到主机的内存中

嘿，怎么有点Mysql的感觉了？扯远了，第一种解决方法又详细地分为了卷挂载和绑定挂载，至于这三者有何区别，且看手册中这张图片：
![](/img/docker/volume1.png)

哈哈，你以为我要开始长篇大论了，别急，让我们先对这张图片有个印象就好，观察到三者的不同就行，我先得设计个实验来验证上面的默认不可持久化！

### lib01

首先，打开终端，(其实docker也提供了非常好用的GUI但我就想用Terminal)，运行一个简单的容器命名为test1,并进入到其中。

```bash
docker run -it --name test1 ubuntu:latest bash
```

在容器内部执行命令新建文件并写入测试内容作为测试数据。

```bash
echo "Test data in container layer" > /test_file.txt
```

检查内容是否成功写入并退出容器。

```bash
cat /test_file.txt
exit
```

使用同一个镜像来新建另一个容器，重复上述步骤查看指定文件是否存在

```bash
docker run -it --name test2 ubuntu:latest bash
ls | grep test_file.txt
```

发现并不存在这个文件，证明当容器不存在时，在可写容器层创建的文件数据不会持久保存。

并且我们也无法从主机上直接访问这个文件，发现无法通过常规的主机文件系统路径找到该文件。

## Volumes

那么我们来看设计师们提供的第一种方法，也是最推荐的方法——Volumes。

如果让我用三句话来总结Volumes:在[docker host]()中开辟一片空间，将容器使用的数据挂载到这片空间中；整个过程以及后续的操作都由docker管理；便于在不同的容器之间共享数据。

形象地解释就是容器将保存的数据从容器内部移到了docker host中，每次需要的时候带上（挂载）就行。

哇哦，这么牛逼?!一次就实现了持久化和共享操作？操作一下看看怎么个事！
![](/img/ys/捂嘴憋笑.webp)

###  lib02

1. 持久性验证

```bash
# 创建名为my_volume的卷
docker volume create --name my_volume
# 运行一个简单的容器并挂载这个卷
docker run -it -v my_volume:/app/data --name test3 ubuntu:latest bash
# 使检查是否能够访问这个卷中的内容
ls /app/data
cat /app/data/test.txt
# 退出容器，从Docker角度检查卷内容
exit
docker volume inspect my_volume
```

显示结果为

```bash
# cat结果
Hello from container
# inspect结果
[
    {
        "CreatedAt": "2024-11-18T09:59:02Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/my_volume/_data",
        "Name": "my_volume",
        "Options": null,
        "Scope": "local"
    }
]
```

重启启动一个新的容器，挂载my_volume卷并检查在不在，步骤除了容器命名不同以外都与上面一致。

最终可以发现，这里的持久性指的就是这个my_volume永久地保存在了docker host中，想要用的时候挂载它就好了！

2. 多个容器共享卷验证

启动两个容器挂载同一个卷，保持着两个容器都处于运行状态，这里我就刚好拿上面两个容器做示范了。

```bash
# 在第一个容器中修改test.txt中的内容，在第二个容器中观察内容是否发生变化
echo "Updated from container 1" >> /app/data/test.txt
# 最终观察到内容为
Hello from container
Updated from container 1
```

显然，这个卷中的内容的更改影响到了两个挂载到它的不同容器（诶，这里面是不是就会存在一些数据冲突并发问题？）

## Bind mounts

哇哦，卷挂载这么厉害啊，那再看看设计者提供的第二种方案：绑定挂载。

如果让我用三句话来总结绑定挂载：将数据文件挂载到宿主机文件系统中，使用宿主机的绝对路径并由宿主机全权管理整个过程。

诶，这和卷挂载的区别不就很明显了嘛，卷挂载放在docker host中，而绑定挂载直接放在了宿主机上。

### lib03

测试绑定挂载

```bash
# 在本机上创建测试文件夹和测试文件
mkdir -p /host_test_dir && cd /host_test_dir
echo "Initial content from host" > host_file.txt
# 挂载这个文件夹启动一个容器,注意，这里必须是在宿主机上的绝对路径
docker run -it -v /$HOME/host_test_dir/:/container_test_dir --name test5 ubuntu:latest bash
# 检查文件
cat /container_test_dir/host_file.txt
```

## tmpfs

最后一种方法自然是将数据保存在宿主机的内存中，但众所周知保存在内存中只是一种临时的解决方案，毕竟宿主机关闭后数据就会消失。

![](/img/ys/药水挥拳.webp)

## 如何选择

那么，在卷挂载和绑定挂载之间该如何选择呢？其实从二者的特点就可以很明显看出docker的设计者想让我们用哪个，没错，一定是卷挂载——因为卷挂载的过程是由docker host管理的，而绑定是由宿主机管理的，这会带来什么影响呢？

卷挂载：

* 其存储位置在 Docker 主机上是被抽象化的。这意味着数据的存储与主机的文件系统结构解耦，即使主机的文件系统布局发生变化，如系统升级或者目录结构调整，卷中的数据依然可以被容器正常访问，因为 Docker 会维护卷的内部结构和数据存储路径。（至于为什么，我们稍后会进行实验验证）
* 卷的生命周期独立于容器。当一个容器停止或被删除时，卷仍然存在，并且可以被挂载到其他容器中，数据不会丢失
* 总的来说就两字：**隔离**

绑定挂载：

* 使得容器对文件的访问依赖于主机的文件系统结构。如果主机上的绑定目录被移动、重命名或者删除，容器内的文件访问就会出现问题。
* 反过来，容器可以直接访问主机的文件系统路径，一个被攻破的容器（例如，容器内的应用存在安全漏洞）可能会利用绑定挂载的权限对主机文件进行未经授权的修改，包括修改重要的系统文件或者其他容器可能依赖的文件

## 关于卷挂载中发现的问题

Hold on,Hold on,我知道看到现在你一定会存在一些小问题，没错我在看官方手册时以及配合AI进行实验验证的过程中也存在了许多问题，比如：

* my_volume创建的路径在哪里？
* 如果我不显式地设置卷挂载呢？会有什么效果？

我们先来回答第二个问题。

### 匿名卷

继续做一个模拟测试实验。

```bash
# 使用-v 但没有指定容器外的挂载路径，只有容器内部的路径
docker run -it -v /app/data --name test6 ubuntu:lat
est bash
# 查看卷列表
docker volume ls
```

![](/img/docker/anonomous_volumes.png)

可以发现除了my_volume这个显示的卷挂载之外还有一个一长串字符组成的卷，这个卷就是匿名卷，是Docker host为我们自动设置的一个随机的、唯一的卷名称。其他的特点与普通卷挂载都是相同的。

### 路径问题

还记得我们之前使用过的`docker volume inspect my_volume`命令得到的路径结果`"Mountpoint": "/var/lib/docker/volumes/my_volume/_data"`

直接在宿主机上找到这个路径不就得了？错误的，要记住卷挂载是保存在docker host之中的，所以在宿主机中一定是找不到这个路径下的文件的。

那么该如何找到这个路径呢？Docker在主机上实际上是基于一个小型的虚拟机来运行的，所以我们得找到这个虚拟机，经过google，我找到了一种[在macos上寻找虚拟机的方法](https://stackoverflow.com/questions/38532483/where-is-var-lib-docker-on-mac-os-x)

```bash
docker run -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh
cd /var/lib/docker/volumes
```

之后就可以在里面查看docker host中的卷以及卷内容了。

![](/img/docker/varlib.png)

关于Docker_Engine中的Storage的相关知识与模拟实验验证就到此结束了，其中也挖了一些以后需要填的坑，日后再见！感谢您抽出宝贵的时间浏览这篇内容。

![](/img/ys/ye.webp)