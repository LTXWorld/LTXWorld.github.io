+++
date = '2025-03-28T14:19:09+08:00'
title = 'K3sEP02——解决RiscV开发板镜像无法拉取问题'
categories = ["云计算"]
tags = ["RISC-V","k3s","pod","适配"]
+++

## 引子

继上次我们验证K3s在虚拟机上的集群部署，我们这次直接在RiscV开发板上实验部署K3s。

值得注意的是，k3s官方支持的架构里面是没有RiscV的，所以有了我们即将开展的适配工作，同时k3s的底层源码大部分都是由Golang构成，而Golang是适配RiscV的，所以我们针对RiscV的K3s移植工作是有可行性的。

本次实验利用了这两篇文章中的配置:[玩转RISCV开发板01-烧录OpenEuler国产镜像](https://www.bfsmlt.top/posts/003li_riscv01/),[玩转RISCV开发板02-配置好容器化环境](https://www.bfsmlt.top/posts/004risc-v_deployment/)，关键在于使用了RiscV+Openeuler的架构系统组合，来自于SIG兴趣小组的23.09的镜像，以及其已经貌似适配好的k3s包。（具体请参考上面的两篇文章）

## 实验环境

- LiCheePi4A 16+256G
- OpenEuler23.09镜像,[下载地址](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/preview/openEuler-23.03-V1-riscv64/lpi4a/)

## 实验过程

我们之前使用`sudo dnf install -y k3s`成功安装了k3及其依赖

```bash
[root@openeuler-riscv64 ~]# sudo dnf install -y k3s
mainline                                        3.8 kB/s | 3.0 kB     00:00    
epol                                            5.4 kB/s | 3.0 kB     00:00    
ceph-user                                       4.6 kB/s | 3.0 kB     00:00    
chromium-user                                   4.4 kB/s | 3.0 kB     00:00    
libreoffice                                     4.6 kB/s | 3.0 kB     00:00    
rust167-user                                    5.8 kB/s | 3.0 kB     00:00    
mesa-supplements                                3.7 kB/s | 3.0 kB     00:00    
oetestsuite                                     5.1 kB/s | 3.0 kB     00:00    
update                                          4.6 kB/s | 3.0 kB     00:00    
Dependencies resolved.
================================================================================
 Package                  Arch     Version                      Repo       Size
================================================================================
Installing:
 k3s                      riscv64  1.24.2+rc1+k3s2-3.oe2303     epol       59 M
Installing dependencies:
 container-selinux        noarch   2:2.163-1.oe2303             mainline   39 k
 k3s-selinux              noarch   1.1.stable.1-1.oe2303        epol       20 k
 policycoreutils          riscv64  3.4-1.oe2303                 mainline  156 k
 selinux-policy           noarch   38.6-4.oe2303                mainline   24 k
 selinux-policy-targeted  noarch   38.6-4.oe2303                mainline  6.8 M

Transaction Summary
================================================================================
Install  6 Packages

Total download size: 66 M
Installed size: 87 M
Downloading Packages:
(1/6): container-selinux-2.163-1.oe2303.noarch.  22 kB/s |  39 kB     00:01    
(2/6): selinux-policy-38.6-4.oe2303.noarch.rpm   13 kB/s |  24 kB     00:01    
(3/6): policycoreutils-3.4-1.oe2303.riscv64.rpm  84 kB/s | 156 kB     00:01    
(4/6): selinux-policy-targeted-38.6-4.oe2303.no 4.1 MB/s | 6.8 MB     00:01    
(5/6): k3s-selinux-1.1.stable.1-1.oe2303.noarch  14 kB/s |  20 kB     00:01    
(6/6): k3s-1.24.2+rc1+k3s2-3.oe2303.riscv64.rpm 4.2 MB/s |  59 MB     00:14    
--------------------------------------------------------------------------------
Total                                           4.2 MB/s |  66 MB     00:15     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Running scriptlet: selinux-policy-targeted-38.6-4.oe2303.noarch           1/1 
  Preparing        :                                                        1/1 
  Installing       : policycoreutils-3.4-1.oe2303.riscv64                   1/6 
  Running scriptlet: policycoreutils-3.4-1.oe2303.riscv64                   1/6 
Created symlink /etc/systemd/system/multi-user.target.wants/restorecond.service → /usr/lib/systemd/system/restorecond.service.

  Installing       : selinux-policy-38.6-4.oe2303.noarch                    2/6 
  Running scriptlet: selinux-policy-38.6-4.oe2303.noarch                    2/6 
  Running scriptlet: selinux-policy-targeted-38.6-4.oe2303.noarch           3/6 
  Installing       : selinux-policy-targeted-38.6-4.oe2303.noarch           3/6 
  Running scriptlet: selinux-policy-targeted-38.6-4.oe2303.noarch           3/6 
  Running scriptlet: container-selinux-2:2.163-1.oe2303.noarch              4/6 
  Installing       : container-selinux-2:2.163-1.oe2303.noarch              4/6 
  Running scriptlet: container-selinux-2:2.163-1.oe2303.noarch              4/6 
  Running scriptlet: k3s-selinux-1.1.stable.1-1.oe2303.noarch               5/6 
  Installing       : k3s-selinux-1.1.stable.1-1.oe2303.noarch               5/6 
  Running scriptlet: k3s-selinux-1.1.stable.1-1.oe2303.noarch               5/6 
  Installing       : k3s-1.24.2+rc1+k3s2-3.oe2303.riscv64                   6/6 
  Running scriptlet: selinux-policy-targeted-38.6-4.oe2303.noarch           6/6 
  Running scriptlet: container-selinux-2:2.163-1.oe2303.noarch              6/6 
  Running scriptlet: k3s-selinux-1.1.stable.1-1.oe2303.noarch               6/6 
  Running scriptlet: k3s-1.24.2+rc1+k3s2-3.oe2303.riscv64                   6/6 
  Verifying        : container-selinux-2:2.163-1.oe2303.noarch              1/6 
  Verifying        : policycoreutils-3.4-1.oe2303.riscv64                   2/6 
  Verifying        : selinux-policy-38.6-4.oe2303.noarch                    3/6 
  Verifying        : selinux-policy-targeted-38.6-4.oe2303.noarch           4/6 
  Verifying        : k3s-1.24.2+rc1+k3s2-3.oe2303.riscv64                   5/6 
  Verifying        : k3s-selinux-1.1.stable.1-1.oe2303.noarch               6/6 

Installed:
  container-selinux-2:2.163-1.oe2303.noarch                                     
  k3s-1.24.2+rc1+k3s2-3.oe2303.riscv64                                          
  k3s-selinux-1.1.stable.1-1.oe2303.noarch                                      
  policycoreutils-3.4-1.oe2303.riscv64                                          
  selinux-policy-38.6-4.oe2303.noarch                                           
  selinux-policy-targeted-38.6-4.oe2303.noarch                                  

Complete!
```

### 验证部署Server节点

接下来我们需要验证部署Server节点，由于之前我们安装好的k3s只是一个运行脚本，所以我们需要执行 `INSTALL_K3S_SKIP_DOWNLOAD=true k3s-install.sh`
并执行`kubectl get nodes`检查集群中的节点。

```bash
[root@openeuler-riscv64 ~]# INSTALL_K3S_SKIP_DOWNLOAD=true k3s-install.sh
[INFO]  Skipping k3s download and verify
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
[ 4166.015532] bridge: filtering via arp/ip/ip6tables is no longer available by default. Update your scripts to load br_netfilter if you need this.
[ 4166.085229] Bridge firewalling registered
[ 4436.489721] IPVS: Registered protocols (TCP, UDP)
[ 4436.501236] IPVS: Connection hash table configured (size=4096, memory=32Kbytes)
[ 4436.507135] IPVS: ipvs loaded.
[ 4438.223653] IPVS: [rr] scheduler registered.
[root@openeuler-riscv64 ~]# kubectl get nodes
NAME                STATUS   ROLES                  AGE     VERSION
openeuler-riscv64   Ready    control-plane,master   3m37s   v1.24.2+k3s-
```

这个过程中跳过了k3s的下载和架构验证，创建了系统 kubectl, crictl,ctr 三个命令的系统链接到 k3s.
创建了 kill all，uninstall 脚本，创建了环境变量，服务文件。

需要提醒的是，此时我们只有一个节点一个开发板，而在 K3s 的源码中默认**Server既是Server又是Agent**，这样才能在单机的情况下运行。

之后我们验证其关于 pod,容器等具体功能是否支持。

我们可以写一个简单的 yaml 文件，指定使用`riscv64/busybox:latest`这个镜像，执行`kubectl apply -f b1.yaml`进行部署
可以发现其创建成功了，但是在执行`kubectl get pods`时显示的状态一直是`ContainerCreated`
我们需要使用`kubectl describe pod b1`来观察这个pod创建的具体过程。

ps:由于本记录没有与实验同步进行（这是我需要吸取的一个教训）所以没有相应的截图可以贴出。
这里拿一个类似的情况举例,发现`pause`镜像无法拉取导致其他 Pod 无法正常启动.

```bash
 Warning  FailedCreatePodSandBox  38s (x129 over 108m)  kubelet  (combined from similar events): Failed to create pod sandbox: rpc error: code = Unknown desc = failed to get sandbox image "carvicsforth/pause:v3.10-v1.31.1": failed to pull image "carvicsforth/pause:v3.10-v1.31.1": failed to pull and unpack image "docker.io/carvicsforth/pause:v3.10-v1.31.1": failed to resolve reference "docker.io/carvicsforth/pause:v3.10-v1.31.1": failed to do request: Head "https://registry-1.docker.io/v2/carvicsforth/pause/manifests/v3.10-v1.31.1": dial tcp 208.101.60.87:443: i/o timeout
```

可以发现其成功 Scheduled 了，但是拉取镜像 pause:3.6 时遇到了 manifest 问题，其实就是pause镜像并不支持 RiscV架构.
于是我们就来深入研究一下 pause 镜像这个东西。

> Pod需要一个中间容器Infa来实现，而这个Infra容器正是pause容器

Pause在K8s的源码位置 `/kubernetes/build/pause/linux/pause.c`，其作用就是

- 无限循环调用 `pause` 系统调用并休眠直到收到信号。
- 作为 PID 为1的进程，专一于维护命名空间和进程管理。

故我们需要先解决这个 pause 镜像的适配问题，由于其源码就是一段C代码而已，所以我们直接尝试在开发板上进行镜像的适配。

### 适配Pause镜像

如果没有安装编译工具链，请先安装`dnf install -y gcc`

将上述 pause.c文件下载或传输到开发板上,这里使用ssh相关命令

`scp -r 本机文件路径 目标主机名@IP:目标路径 `

以下步骤参考整合于 Deepseek 与 ChatGpt 的回答。

**1.直接编译现有代码**

```bash
gcc -static -Os -o pause pause.c
```

**2.检查并测试**

```bash
ldd ./pause
# 测试版本显示
./pause -v
```

**3.容器化使用**

```dockerfile
FROM scratch
COPY pause /pause
ENTRYPOINT ["/pause"]
```

**4.测试容器**

```bash
# 构建镜像
docker build -t my-pause .

# 运行测试
docker run --rm -it --name test-pause my-pause

# 在另一个终端发送停止信号
docker stop test-pause  # 应看到正常退出
```

**5.docker保存镜像为tar文件**

```bash
docker save my-pause > pause-riscv.tar
```

**6.在k3s节点上加载镜像**

```bash
# 先查看当前k3s使用的containerd命名空间,我的结果是k8s.io
k3s ctr namespace ls
# 再加载
k3s ctr images import pause-riscv.tar
# or
k3s ctr -n k8s.io images import pause-riscv.tar
```

**7.将此镜像作为本地镜像**

```bash
# 如果没有这个文件夹，自行创建
cd /var/lib/rancher/k3s/agent/images
mkdir -p /var/lib/rancher/k3s/agent/images
# 放置我们自己的pause镜像
cp ./pause-riscv.tar /var/lib/rancher/k3s/agent/images
```

**8.将此镜像打标签以适应原本拉取时所要的镜像名称**

```bash
# 先查看当前我们制作的镜像名称
ctr images ls
# 重新打标签 后面的名称是之前k3s拉取时拉取失败的镜像名称 
ctr -n k8s.io images tag <上一条命令的名称> docker.io/rancher/mirrored-pause:3.6
```

**9.再使用 crictl,ctr 命令检查镜像**

将这两个命令理解为docker命令即可，用来管理镜像。

```bash
ctr --namespace=k8s.io images list | grep pause
crictl images | grep pause
```

**10.重新尝试busybox镜像**

```bash
kubectl apply-f b1.yaml
# 再用crictl命令,使用debug标签观察其过程
crictl --debug pull riscv64/alpine
# 最后检查pod是否运行成功
kubectl get pods -o wide
```

最终发现busybox和alpine镜像均可以成功拉取，不会再出现pause镜像不适配的问题。

其他镜像可能会因为网络问题，适配问题无法拉取成功，这是我们后续需要进行的工作。

## 回到源码

回到源码之中，在`cli/cmds/agent.go`源码中，关于pause-image命令的代码如下

```go
PauseImageFlag = &cli.StringFlag{
	Name:        "pause-image",
	Usage:       "(agent/runtime) Customized pause image for containerd or docker sandbox",
	Destination: &AgentConfig.PauseImage,
	Value:       "rancher/mirrored-pause:3.6",
}
```

所以在启动agent节点时可以直接在命令行中指定

```bash
sudo k3s agent \
  --pause-image=your-repo/your-pause:3.6-riscv64 \
  --server=https://<K3s-Server-IP>:6443 \
  --token=<Your-Token>
```

源码中关于pause的构建Dockefile如下：

```dockefile
ARG BASE
FROM ${BASE}
ARG ARCH
ADD bin/pause-linux-${ARCH} /pause
USER 65535:65535
ENTRYPOINT ["/pause"]
```

可供我们构建时参考。

## 总结

至此，我们完成在 RiscV+Openeuler 开发板上 k3s 的 Server 节点部署工作，并成功解决了其拉取 pause 镜像出现的架构不适配问题。
我们采用的方法是本地自行构建适合 riscv 的 pause 镜像并让 k3s 使用这个镜像启动.

后续目标再仔细研究 k3s/k8s 源码中有关于 Pod 创建的过程，看看其对 pause的具体应用在哪里.
如何能够将 pause 这部分的缺陷从源码的部分补充完整，可以不再需要直接从本地构建，而是可以直接部署。

## 更新(5.8)

从大佬的思路中得到启发,我们可以修改源码中的`pkg/cli/cmds/const_linux.go`文件的 `DefaultPauseImage  = "rancher/mirrored-pause:3.6"` 
换为我们自己仓库的地址,例如`carvicsforth/pause:v3.10-v1.31.1`(这是大佬的地址),这样就可以避免了本地构建.