+++
date = '2024-11-29T21:07:21+08:00'
title = '玩转RISCV开发板02-配置好容器化环境'
categories = ["硬核技术"]
tags = ["RISC-V","设置环境","docker","git","k3s"]
+++

# 前言

这里照例列出所用设备配置

* macbookair
* LicheePi4A,16+128

![](/img/ys/晚上好兄弟们.jpg)

# 基础配置

如果你看完了上篇文章，那么一定成功连接到了wifi，那么我们就先用ssh从主机的终端连接到开发板（毕竟我只有一个显示器，开发板你不配）

先打开开发板，并连接显示器，按照上一节的做法查看wifi是否连接，如果使用的nmcli这个命令它会自动连接的。接着使用`hostname -I`查看当前IP地址。拿着这个IP地址使用ssh去连接开发板。

```bash
# 将IP换为具体的ip地址
ssh openeuler@IP
```

接下来，我们开始设置最基础的配置，例如密码，镜像源，以及一些小工具等等。

## 密码与换源

首先，更改这个复杂的初始密码openEuler12#$,输入`passwd`，修改当前用户的密码（注意需要至少3个不同的字符类型）

其次，关于包管理软件，使用dnf和yum都可以，但是推荐使用dnf，毕竟是yum的上游版本；关于是否需要换源，可以先去`/etc/yum.repos.d/openEuler.repo`这个文件中查看自己当前使用的源地址，其实里面的内容就是我们在01中下载镜像的地址。这是兴趣小组维护的一个很棒的地址，所以我认为没必要更换源。

如果你想更换，这是openEuler官网所列出的[镜像源](https://www.openeuler.org/zh/mirror/list/),可以选择并替换掉上述文件中的链接内容。

## 安装git

这个不用赘述`sudo dnf install git`

如果遇到从开发板下载github上的东西特别慢的情况，可以使用这个网站`https://doget.nocsdn.com/#/`托管的国内地址来下载，速度不错。

## tmux分屏

这个软件如果你是一个linux或者macos的“老玩家”，相信你并不陌生。有了他我们就可以不再打开一个又一个的终端窗口，而是优雅的进行分屏操作。

惊喜的是，软件源中居然存在这个软件，向兴趣小组的成员致敬。🫡

![](/img/ys/ye.webp)

```bash
sudo dnf install tmux
```

下载成功。
![](/img/riscv/tmux.png)

简单地试用一下，关于tmux的个性化配置，请参考[这篇文章]。（我先挖个坑，绝对会填上的）😭

```bash
tmux new -s "riscv"
```

输入CTRL+b作为前导键，再按下c新建一个窗口，会得到这样的效果
![](/img/riscv/tmux2.png)

其实我现在在macos使用ssh的情况下已经是tmux套tmux的情况了😁

![](/img/jb/coffee.webp)

其他的操作请参考[这篇文章]。

# 配置Golang环境

首先，来到Golang的官网[release界面](https://go.dev/dl/)找到能够适配RISC-V的Golang版本，这里我使用的是[go1.22.9.linux-riscv64.tar.gz](https://go.dev/dl/go1.22.9.linux-riscv64.tar.gz),可以看到其支持的**操作系统为Linux,架构为riscv64。**

为什么不选择最新的go1.23?因为还没有适配riscv的包。

```bash
# 使用该命令下载对应压缩包
curl -O https://go.dev/dl/go1.22.9.linux-riscv64.tar.gz
# 如果遇到下载文件的错误，比如意外地下载为了其他格式，建议使用以下命令
curl -L -O https://go.dev/dl/go1.22.9.linux-riscv64.tar.gz
#下载成功后，可以用以下命令检查文件格式
file go1.22.9.linux-riscv64.tar.gz

# 对软件包进行解压
tar -zxvf go1.22.9.linux-riscv64.tar.gz
# 配置golang的系统环境
vi ~/.bashrc
# 在其中加入下面这两行之一
export PATH=$PATH:~/go/bin
export PATH=$PATH:$HOME/go/bin
# 或者你可以聪明地一步到位
echo 'export PATH=$PATH:~/go/bin' >> ~/.bashrc
# 重新加载
source ~/.bashrc
# 检查是否成功
go env
```

如果成功看到一系列的go的环境变量，就说明我们就将golang环境配置好了。

接下来可以编写一个简单的main.go文件进行测试

```go
package main
import "fmt"
func main() {
  fmt.Println("Hello, RV64")
}
```

`go run main.go`

**ps**:如果在执行上面的代码过程中遇到`warning: GOPATH set to GOROOT (/home/openeuler/go) has no effect`这个问题，意味着他在提醒你使用module进行管理项目的依赖关系，我们这里只是简单的测试一下，后期遇到的golang项目会使用mod进行管理。

>GOPATH must be set to get, build and install packages outside the standard Go tree.

具体原因在于GOPATH是工作目录，用于存放Go项目的源代码、依赖包；而GOROOT是Go语言的安装目录，二者当然最好要不一样的路径。来自于[Stackoverflow的解释](https://stackoverflow.com/questions/22877775/gopath-must-not-be-set-to-goroot-why-not)

# 配置docker

首先，下载docker engine,可以发现，官网中的列表中是没有Openeuler下的riscv64架构的适配的。

![](/img/riscv/dockerengine.png)

所以我们直接采用dnf包管理的方式下载，(在这里对于为Openeuler-riscv64架构进行docker适配的人致以深刻的敬意，这也许就是真正的伟大的程序员吧）

![](/img/shu/躺平.webp)

```bash
sudo dnf update
sudo dnf install -y docker
# 检查docker的运行状态
sudo systemctl status docker
# 如果没有运行，需要启动
sudo systemctl start docker
```

接下来去sudo化配置，见[官方文档](https://docs.docker.com/engine/install/linux-postinstall/)

接下来进行最重要的换源，否则镜像也无法拉取成功。目前国内还可以使用的Docker镜像见此[链接](https://www.wangdu.site/course/2109.html)

```bash
# 修改
vi /etc/docker/daemon.json
# 如果没有这个文件，自己手动新建（我当时只有一个key.json文件）
# 将下面内容复制进去
{
      "registry-mirrors": [
        "https://dockerpull.org",
        "https://docker.1ms.run",
        "https://docker.linkedbus.com"
    ]
}

# 加载配置并重启
sudo systemctl daemon-reload
sudo systemctl restart docker
# 尝试拉取镜像
docker pull busybox
docker run hello-world
```

看到"Hello from Docker"意味着拉取镜像成功。你也许会疑惑，为什么可以成功？镜像也应该与架构相关啊，使得，hello-world这个镜像正是适配了riscv64架构，不过其他的镜像就没有那么好运了。关于某个镜像是否支持某个架构可以在[dockerhub](https://hub.docker.com/)中的具体镜像页面内查看。

比如我们尝试一下nginx这个镜像`docker pull nginx`，可以发现他没有通过上面设置的仓库而是去官方找镜像，原因由chatgpt给出:

>由于您是在 RISC-V 环境（openEuler-riscv64）下运行，可能镜像加速器的缓存中并没有 RISC-V 架构的 nginx 镜像。因此 Docker 会从官方仓库拉取适配的镜像。

就算是你强制使用换源后的仓库`docker pull dockerpull.org/nginx`，也会提示你`no matching manifest for unknown in the manifest list entries`意味着在镜像仓库中并没有合适的资源。

目前在dockerhub中有关于riscv的镜像一共有[官方的35个仓库](https://hub.docker.com/u/riscv64),这里看到了redis，让我们试一试`docker pull riscv64/redis`，很遗憾，好像还是因为镜像站中没有收录。

最后再尝试一个alpine，其支持riscv64`docker pull alpine`

![](/img/riscv/alpine.png)

最后，关于镜像适配问题，这也是未来的工作方向之一吧，可真是任重而道远啊——总不能让我给每个镜像都来一遍riscv架构的适配吧？这是我这种鼠鼠能做的？

![](/img/ys/药水挥拳.webp)

# 配置K3s

来到k3s的官网使用` curl -sfL https://get.k3s.io | sh -`官方命令安装会显示`[ERROR]  Unsupported architecture riscv64`可以得知官方还并没有真正支持riscv架构。那么怎么办？

谷歌了一圈发现[这里的内容](https://github.com/CARV-ICS-FORTH/kubernetes-riscv64)值得一看，专门在说k8s和k3s向riscv适配的，里面也给出了具体的操作指南。这里我们先试试SIG（兴趣小组）的源中是否存在k3s。

其实原理上来讲，与docker一样源代码都是完全由golang编写的，而golang的官方正如我们上面所说，已经支持了riscv，所以迁移是一定可以实现的（但愿）。🧐

```bash
sudo dnf install k3s
```

嘿，您猜怎么着，还真有！再次respect！让我们看看这次都下载了什么

![](/img/riscv/k3sinstall.png)

> `container-selinux`提供了通用的容器安全策略，而`k3s-selinux`提供了针对k3s的特定安全策略，确保k3s在运行时的安全性

那么下载过后如何执行呢？按道理来说，得`systemctl status k3s`这样的命令（与docker类似）来操作，但是你直接操作时会发现报错`unit k3s service not found`并不存在。这该怎么办呢？

![](/img/ys你说什么.webp)

在我Google了一圈发现这位大佬的[适配工作](https://github.com/carlosedp/riscv-bringup)后，我发现刚才下载的k3s是不是在`/usr/local/bin/k3s`这个路径下啊？

于是去这个路径寻找，果然找到了，`./k3`执行这个文件，执行成功了。

![](/img/riscv/k3srun.png)

关于k3s的具体操作我们以后的文章再说。

# 总结与展望

没什么好总结的了，下一章进行k3s在riscv架构上的初步尝试，写几个golang小程序部署成节点试着管理一下。
可真是Life's struggle啊

![](/img/shu/躺平.webp)