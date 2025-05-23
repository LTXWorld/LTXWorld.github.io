+++
date = '2025-04-19T19:13:50+08:00'
title = 'K3sEP06——从issues上得到的可能尝试'
+++

## 引子

在[K3sEP04](https://www.bfsmlt.top/posts/020k3sep04%E5%BC%80%E5%8F%91%E6%9D%BF%E4%B8%8Ak3s%E5%AD%98%E5%9C%A8%E7%9A%84%E9%97%AE%E9%A2%9802/)中最后的总结中我们提到，反思了一下我们的移植过程，开始从k3s的结构看起，再到k8s的书籍，再到重新看二者的架构设计不同点，再到对应的命令，再追踪到k3s的源码，最后再开发板上适配的时候解决出现的一系列问题，我们处理移植问题的思路首先是有问题的。

还是引用一句话，大致的意思是，“你所遇到的问题100%都在别人的身上发生过，这意味着只要google，就一定可以找到答案。”但是我在这其中难道没有google过吗？我也不清楚为什么我没有发现对应的issue，直到最近，我才在github中找到了k3s关于支持riscv的相关issue，这证明自己处理此类问题的方法很有问题。

虽然我也可以甩锅到带我的老师没有指引我，但实际上这是一种解决问题的能力欠缺的表现，就像我今天读到的一篇文章，“编程思维不是说你能写出多么漂亮的代码，而是考验你如何快速高效解决问题”，是的，从这件事情上我意识到其实自己这方面的能力十分欠缺。

说回来，在[这个issue](https://github.com/k3s-io/k3s/issues/7151)中我发现了一个最有意思的问题，在2024-08-30，一位开发者说他只通过三条命令就在riscv64的机器上安装好了k3s并能够顺利运行。

![01](/img/riscv/issue01.png)

可以看到这位开发者和我的思路一致，都是自行构建解决了pause镜像的问题，所以我们根据他们的说法来试试吧，从k3s的源码构建，而不是简单的`dnf install k3s`

## 从源码构建可运行的k3s二进制文件

```bash
cd k3s
./script/download
./script/build
./script/package-cli
```

一共就是这三条命令，就可以构建出可执行的k3s文件，注意执行顺序不可更改。

接下来我来排一下可能会遇到的坑。

### yq未安装

yq是一个处理YAML文件的命令行工具,K3s构建脚本中使用它来解析yaml文件，所以需要提前安装yq。如果`dnf install yq`显示包管理器中没有yq的话，我们就需要从源码进行构建。

```bash
# 需要安装 go 编译环境
git clone https://github.com/mikefarah/yq.git
cd yq/v4
go build -o yq .
sudo mv yq /usr/local/bin/
```

或者直接去[官方的release](https://github.com/mikefarah/yq/releases)中查看有没有编译好的，发现其实是有的，可以直接进行下载使用

```bash
wget https://github.com/mikefarah/yq/releases/download/v4.44.6/yq_linux_riscv64.tar.gz
tar -zvxf yq_linux_riscv64.tar.gz
mv yq_linux_riscv64.tar.gz yq
chmod +x yq
mv yq /usr/local/bin
yq --version
```

### 有可能出现的源码错误

关于这个错误我也很疑惑，我并没有在开发板上修改过源码的内容，但是显示

```bash
-w -s ' -o bin/k3s ./cmd/server
# github.com/k3s-io/k3s/pkg/agent
pkg/agent/run.go:110:53: syntax error: unexpected name fmeEndpoint, expected {
```

证明源码中有地方出错了，我对比之下发现确实多了个空格，如果有遇到相同问题的朋友欢迎交流。

### libseccomp未安装

与yq不一样的是，这是linux系统级别的底层库，如果未安装的话其大概率是在包管理器中的，我们可以先检查一下是否存在。

```bash
dnf list libseccomp\*
Last metadata expiration check: 1:23:59 ago on 2025年04月19日 星期六 18时37分42秒.
Installed Packages
libseccomp.riscv64                                           2.5.4-1.oe2309                               @OS
libseccomp-devel.riscv64                                     2.5.4-1.oe2309                               @OS
Available Packages
libseccomp-debuginfo.riscv64                                 2.5.4-1.oe2309                               OS
libseccomp-debugsource.riscv64                               2.5.4-1.oe2309                               OS
libseccomp-help.noarch                                       2.5.4-1.oe2309                               OS
libseccomp-help.noarch                                       2.5.4-1.oe2309                               EPOL
```

这里显示我已经安装好了，如果你没有安装的话，使用`dnf install libseccomp-devel.riscv64`进行安装。

其他缺失的类似库也一样，我们都可以先list检查其是否存在然后再选择安装还是自行编译。（毕竟这是一个比较新的系统）

### 结果

最终执行完毕之后结果如下:

```bash
+ go build -tags urfave_cli_no_docs -buildvcs=false -ldflags '
    -X github.com/k3s-io/k3s/pkg/version.Version=v1.32.3+k3s-76c5c770
    -X github.com/k3s-io/k3s/pkg/version.GitCommit=76c5c770
    -w -s
 -extldflags '\''-static'\''' -o dist/artifacts/k3s-riscv64 ./cmd/k3s
+ stat dist/artifacts/k3s-riscv64
  文件：dist/artifacts/k3s-riscv64
  大小：80478360        块：157192     IO 块大小：4096   普通文件
设备：179,3     Inode: 661710      硬链接：1
权限：(0755/-rwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
访问时间：2025-04-19 19:04:50.007082376 +0800
修改时间：2025-04-19 19:04:50.375090740 +0800
变更时间：2025-04-19 19:04:50.375090740 +0800
创建时间：2025-04-19 19:04:50.007082376 +0800
+ ./scripts/build-upload dist/artifacts/k3s-riscv64 76c5c770b21e101f00a32445d3e6fdc325d563b5
AWS_SECRET_ACCESS_KEY is not set
```

最终的二进制文件是`./dist/artifacts/k3s-riscv64`，我们可以在后面加上server和agent来分别执行这个二进制文件。

## 执行过程中存在的问题

当然执行并不是一蹴而就的，会存在一系列的问题（谁能想到我第一次执行成功之后的就一直在失败）

### Golang的版本不匹配问题

如果我们真正执行起来会发现出现 Golang 的版本不匹配问题

```bash
./dist/artifacts/k3s-riscv64 server
INFO[0000] Acquiring lock file /var/lib/rancher/k3s/data/.lock
INFO[0000] Preparing data dir /var/lib/rancher/k3s/data/ce5fac18afd90ce4ba321d56c827d30b3b3de0cbe5c654dbb397bf62a178878e
FATA[0000] Failed to validate golang version: incorrect golang build version - kubernetes v1.32.3 should be built with go, runtime version is go1.23.3
```

意味着当前 k8s 需要的 Golang 版本不是我们目前的 go1.23.3，所以在哪里查找其需要的特定 go 版本呢？

我 google 并没有得到准确的答案，gpt 告诉我在这个 [commit](https://github.com/kubernetes/kubernetes/commit/ae0ec29cbf463ad5fffb67907546c98f1d667855)中，其修改了 v1.32.3版本的 k8s 所需版本为 go1.23.6

其实还可以从构建脚本中发现其所需要的版本，在`build`脚本中的这两句

```bash
DEPENDENCIES_URL="https://raw.githubusercontent.com/kubernetes/kubernetes/${VERSION_K8S}/build/dependencies.yaml"
VERSION_GOLANG="go"$(curl -sL "${DEPENDENCIES_URL}" | yq e '.dependencies[] | select(.name == "golang: upstream version").version' -)
```

就是在这个 `dependencies.yaml` 文件中寻找 Golang 的版本，我们打开这个文件发现 v1.32.3的 K8s 所需的 Golang 版本为 1.23.6，与上面 commit 中的版本相同，验证了我们的想法。

```yaml
# Golang
  - name: "golang: upstream version"
    version: 1.23.6
```

所以我们构建就要使用 go1.23.6，想想也挺奇怪的，明明 k3s 源码的 go.mod 中指明了 go1.23.3,但是其依赖的 k8s 中的内容却要 go1.23.6???

### 奇怪的downloading

一旦进入源码目录执行一切 go 开头的命令就会出现 `go: downloading go1.23.3 (linux/riscv64)`，我的 go 本地本来就有啊，为什么他还是会去 download 呢？并且去非 k3s 源码的其他目录执行 `go version`类似命令，正常输出。

原因如下：在 Go 里面，如果当前目录下有 go.mod 文件，不管你执行什么 go 命令，它都会默认开启 module模式。

- 检查 go.mod 和 go.sum
- 有需要时自动联网去拉缺失的模块

## 解决可能出现的问题

### 尝试切换版本

我们推倒重来，默认使用Golang版本为1.23.6,即构建版本为1.23.6；使用时切换到1.22or1.21。
安装时提前下载好对应版本的Golang安装包。

```bash
rm -rf /usr/local/go
tar -C /usr/local -xzf go1.23.6.linux-riscv64.tar.gz
vi ~/.bashrc
# Go environment
export GOROOT=/usr/local/go
export PATH=$GOROOT/bin:$PATH

export GOPATH=$HOME/go
export PATH=$GOPATH/bin:$PATH

export GOPROXY=https://goproxy.cn,direct
#
source ~/.bashrc
go version
# 将另一个版本放到另一个目录下
mkdir -p /opt
tar -C /opt -xzf go1.22.9.linux-riscv64.tar.gz
mv /opt/go /opt/go1.22.9
vi ~/.bashrc
# 在最后添加转换脚本
usego() {
  if [ "$1" = "1.23.6" ]; then
    export GOROOT=/usr/local/go
  elif [ "$1" = "1.22" ]; then
    export GOROOT=/opt/go1.22.9
  else
    echo "Usage: usego [1.23.6|1.22]"
    return 1
  fi
  export PATH=$GOROOT/bin:$PATH
  echo "Switched to Go version: $(go version)"
}
#
source ~/.bashrc
```

之后我们可以执行`usego 1.23.6`和`usego 1.22`来回切换开发板上的Golang版本了。

执行构建`./script/download`

### Download遇到网络问题

如果开发板上遇到网络问题（即使我添加了代理也无法解决），我从 gpt 上得到的建议是可以在主机上进行 download 操作，因为这一步只是下载所需的各种依赖，并不是编译过程，所以不太涉及到架构问题。

只需要在 download 前引入 riscv64 的临时环境变量，让其安装的依赖适合 riscv64。

```bash
export GOARCH=riscv64
export ARCH=riscv64
export OS=linux

./scripts/download
# 然后拷贝build目录到开发板的k3s源码上
scp build/ root@ip:/root/k3s
```

这样就可以免于在开发板上进行直接 download，从而避免长时间的等待。

## 探究为什么可以成功

### download

这里我们列出其脚本内容，不算太多可以分析一下。分析我都写在每行命令的后面方便查看。

```bash
#!/bin/bash # 指定以bash启动脚本

set -ex # 设置如果任何命令返回非零退出状态，立即退出；打印执行的每个命令及其参数，便于调试

cd $(dirname $0)/.. # 切换到脚本所在目录的父目录

. ./scripts/version.sh # 执行version.sh脚本，里面定义了版本信息

CHARTS_URL=https://k3s.io/k3s-charts/assets # 定义各种URL和目录路径变量
CHARTS_DIR=build/static/charts
RUNC_DIR=build/src/github.com/opencontainers/runc
CONTAINERD_DIR=build/src/github.com/containerd/containerd
HCSSHIM_DIR=build/src/github.com/microsoft/hcsshim
DATA_DIR=build/data
export TZ=UTC

umask 022 # 设置新创建文件的默认权限（755目录/644文件）
rm -rf ${CHARTS_DIR} # 删除之前的目录并创建空的新目录
rm -rf ${RUNC_DIR}
rm -rf ${CONTAINERD_DIR}
rm -rf ${HCSSHIM_DIR}
mkdir -p ${CHARTS_DIR}
mkdir -p ${DATA_DIR}

case ${OS} in # 如果系统是linux就会下载指定版本，指定架构的runc(底层容器运行时，负责运行 Pod 中的容器)以及k3s-root并解压
  linux)
    git clone --single-branch --branch=${VERSION_RUNC} --depth=1 https://github.com/k3s-io/runc ${RUNC_DIR}
    curl --compressed -sfL https://github.com/k3s-io/k3s-root/releases/download/${VERSION_ROOT}/k3s-root-${ARCH}.tar | tar xf -
    ;;
  windows) # 如果是windows
    git clone --single-branch --branch=${VERSION_HCSSHIM} --depth=1 https://github.com/microsoft/hcsshim ${HCSSHIM_DIR}
    ;;
  *) # 如果是其他系统就会直接输出错误信息并退出
    echo "[ERROR] unrecognized operating system: ${OS}"
    exit 1
    ;;
esac # 注意这里case in的用法

git clone --single-branch --branch=${VERSION_CONTAINERD} --depth=1 https://${PKG_CONTAINERD_K3S} ${CONTAINERD_DIR} # 克隆指定版本的containerd（容器运行时）仓库，使用depth=1的浅克隆只获取最新提交

# 从K3s charts仓库下载每个chart文件到构建目录
for CHART_FILE in $(grep -rlF HelmChart manifests/ | xargs yq eval --no-doc .spec.chart | xargs -n1 basename); do
  CHART_NAME=$(echo $CHART_FILE | grep -oE '^(-*[a-z])+')
  curl -sfL ${CHARTS_URL}/${CHART_NAME}/${CHART_FILE} -o ${CHARTS_DIR}/${CHART_FILE}
done
```

从这个 build 脚本可以看出其下载了以下东西:

- runc+k3s-root(linux) / hcsshim(windows) 
- containerd(底层是runc)
- HelmChart(k8s中类似于dnf的包管理器)

并最终将这些内容放在 build 目录下，其中会产生 src, static, data目录。

可以发现，这里的 k3s-root-riscv64是关键，我在 k3s 相关的 [issue](https://github.com/k3s-io/k3s/pull/7778) 中找到了有一位开发者提供了对其的 commit 并得到了 k3s-root 的接受。



### build

接下来我们分层次介绍一下build脚本。

```bash
#!/bin/bash
set -e -x

cd $(dirname $0)/..

. ./scripts/version.sh

GO=${GO-go} # 默认使用 `go` 命令，但允许通过环境变量覆盖

PKG="github.com/k3s-io/k3s"
PKG_CONTAINERD="github.com/containerd/containerd"
PKG_CRICTL="github.com/kubernetes-sigs/cri-tools/pkg"
PKG_K8S_BASE="k8s.io/component-base"
PKG_K8S_CLIENT="k8s.io/client-go/pkg"
PKG_CNI_PLUGINS="github.com/containernetworking/plugins"
PKG_KUBE_ROUTER="github.com/cloudnativelabs/kube-router/v2"
PKG_CRI_DOCKERD="github.com/Mirantis/cri-dockerd"
PKG_ETCD="go.etcd.io/etcd"

buildDate=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
```

前面相同，后面定义了各个依赖组件的 Go 模块路径，用于后续的版本信息注入。

```bash
VERSIONFLAGS="
    -X ${PKG}/pkg/version.Version=${VERSION}
    -X ${PKG}/pkg/version.GitCommit=${COMMIT:0:8}
    -X ${PKG}/pkg/version.UpstreamGolang=${VERSION_GOLANG}

    -X ${PKG_K8S_CLIENT}/version.gitVersion=${VERSION}
    -X ${PKG_K8S_CLIENT}/version.gitCommit=${COMMIT}
    -X ${PKG_K8S_CLIENT}/version.gitTreeState=${TREE_STATE}
    -X ${PKG_K8S_CLIENT}/version.buildDate=${buildDate}

    -X ${PKG_K8S_BASE}/version.gitVersion=${VERSION}
    -X ${PKG_K8S_BASE}/version.gitCommit=${COMMIT}
    -X ${PKG_K8S_BASE}/version.gitTreeState=${TREE_STATE}
    -X ${PKG_K8S_BASE}/version.buildDate=${buildDate}

    -X ${PKG_CRICTL}/version.Version=${VERSION_CRICTL}

    -X ${PKG_CONTAINERD}/version.Version=${VERSION_CONTAINERD}
    -X ${PKG_CONTAINERD}/version.Package=${PKG_CONTAINERD_K3S}

    -X ${PKG_CNI_PLUGINS}/pkg/utils/buildversion.BuildVersion=${VERSION_CNIPLUGINS}
    -X ${PKG_CNI_PLUGINS}/plugins/meta/flannel.Program=flannel
    -X ${PKG_CNI_PLUGINS}/plugins/meta/flannel.Version=${VERSION_FLANNEL_PLUGIN}+${VERSION_FLANNEL}
    -X ${PKG_CNI_PLUGINS}/plugins/meta/flannel.Commit=HEAD
    -X ${PKG_CNI_PLUGINS}/plugins/meta/flannel.buildDate=${buildDate}

    -X ${PKG_KUBE_ROUTER}/pkg/version.Version=${VERSION_KUBE_ROUTER}
    -X ${PKG_KUBE_ROUTER}/pkg/version.BuildDate=${buildDate}

    -X ${PKG_CRI_DOCKERD}/cmd/version.Version=${VERSION_CRI_DOCKERD}
    -X ${PKG_CRI_DOCKERD}/cmd/version.GitCommit=HEAD
    -X ${PKG_CRI_DOCKERD}/cmd/version.BuildTime=${buildDate}

    -X ${PKG_ETCD}/api/v3/version.GitSHA=HEAD
"
```

版本信息注入——使用 -X 标志向 Go 二进制文件注入版本信息，包括k3s版本，Git提交哈希，组件版本等。

```bash
if [ -n "${DEBUG}" ]; then
  GCFLAGS="-N -l"
else
  LDFLAGS="-w -s"
fi
```

在调试模式下禁用优化，便于调试

```bash
case ${OS} in
  linux)
    TAGS="$TAGS apparmor seccomp"
    RUNC_TAGS="$RUNC_TAGS apparmor seccomp"

    if [ "$STATIC_BUILD" == "true" ]; then
      STATIC=$'\n-extldflags \'-static -lm -ldl -lz -lpthread\'\n'
      TAGS="static_build libsqlite3 $TAGS"
      RUNC_STATIC="static"
    fi
    if [ "$SELINUX" = "true" ]; then
      TAGS="$TAGS selinux"
      RUNC_TAGS="$RUNC_TAGS selinux"
    fi
    ;;
  windows)
    TAGS="$TAGS no_cri_dockerd"

    if [ "$STATIC_BUILD" == "true" ]; then
      STATIC=$'\n-extldflags \'-static -lpthread\'\n'
      TAGS="static_build $TAGS"
    fi

    export CXX="x86_64-w64-mingw32-g++"
    export CC="x86_64-w64-mingw32-gcc"
    ;;
  *)
    echo "[ERROR] unrecognized opertaing system: ${OS}"
    exit 1
    ;;
esac
```

关于静态构建选项？

```bash
if [ ${ARCH} = armv7l ] || [ ${ARCH} = arm ]; then
    export GOARCH="arm"
    export GOARM="7"
fi

if [ ${ARCH} = s390x ]; then
    export GOARCH="s390x"
fi
```

根据架构设置交叉编译的 GOARCH 变量

```bash
k3s_binaries=(
    "bin/k3s-agent"
    "bin/k3s-server"
    "bin/k3s-token"
    ...
)

containerd_binaries=(
    "bin/containerd-shim"
    "bin/containerd-shim-runc-v2"
    ...
)

for i in "${k3s_binaries[@]}"; do
    if [ -f "$i${BINARY_POSTFIX}" ]; then
        rm -f "$i${BINARY_POSTFIX}"
    fi
done
```

清理旧的 k3s 和 containerd 的二进制文件

```bash
if [ ! -x ${INSTALLBIN}/cni${BINARY_POSTFIX} ]; then
(
    TMPDIR=$(mktemp -d)
    git clone --single-branch --depth=1 --branch=$VERSION_CNIPLUGINS https://github.com/rancher/plugins.git $WORKDIR
    cd $WORKDIR
    rm -rf plugins/meta/flannel
    git clone --single-branch --depth=1 --branch=$VERSION_FLANNEL_PLUGIN https://github.com/flannel-io/cni-plugin.git plugins/meta/flannel
    GO111MODULE=off GOPATH=$TMPDIR CGO_ENABLED=0 "${GO}" build -tags "$TAGS" -gcflags="all=${GCFLAGS}" -ldflags "$VERSIONFLAGS $LDFLAGS $STATIC" -o $INSTALLBIN/cni${BINARY_POSTFIX}
)
fi
```

克隆并构建 CNI 网络插件（有关于 flannel，后续会写文介绍）

```bash
echo Building k3s
CGO_ENABLED=1 "${GO}" build $BLDFLAGS -tags "$TAGS" -buildvcs=false -gcflags="all=${GCFLAGS}" -ldflags "$VERSIONFLAGS $LDFLAGS $STATIC" -o bin/k3s${BINARY_POSTFIX} ./cmd/server

for i in "${k3s_binaries[@]}"; do
    ln -s "k3s${BINARY_POSTFIX}" "$i${BINARY_POSTFIX}"
done
```

构建k3s主程序，Go 编译器 (go build) 编译 `./cmd/server`(主入口程序) 目录下的代码，生成 bin/k3s 二进制文件。

最后构建 containerd 和 runc;

整个流程确保了 K3s 及其所有依赖组件能正确编译并打包成可执行文件。

### package-cli

此脚本用来构建有关于k3s的相关命令。

## 汇总

最终我们将得到的 k3s-riscv64 二进制文件放入 `/usr/local/bin`中并使用 systemd 来控制其状态。具体步骤如下:

```bash
mv k3s-riscv64 k3s
mv /path/to/k3s /usr/local/bin
chmod +x /usr/local/bin/k3s
# 编写k3s.service
vi /etc/systemd/system/k3s.service
# 内容如下，来自于k3s源码中的同样内容
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=-/etc/default/%N
EnvironmentFile=-/etc/sysconfig/%N
EnvironmentFile=-/etc/systemd/system/k3s.service.env
ExecStartPre=/bin/sh -xc '! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service 2>/dev/null'
ExecStart=/usr/local/bin/k3s server
KillMode=process
Delegate=yes
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

之后重新加载配置并使用 systemctl 管理 k3s

```bash
systemctl daemon-reload
systemctl enable k3s
systemctl start k3s
systemctl status k3s
```

启动成功，状态如下：

```bash
k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; disabled; preset: disabled)
    Drop-In: /etc/systemd/system/k3s.service.d
             └─override.conf
     Active: active (running) since Mon 2025-04-28 14:21:55 CST; 50s ago
       Docs: https://k3s.io
```

执行相关命令

```bash
k3s kubectl get node
NAME                STATUS   ROLES                  AGE   VERSION
openeuler-riscv64   Ready    control-plane,master   40m   v1.32.3+k3s-76c5c770-dirty
```

设置软链接，可以省去前面的 k3s 标识。

```bash
ln -sf /usr/local/bin/k3s /usr/local/bin/kubectl
```

最终将上述操作汇总为`k3s-init.sh`,与 k3s-riscv64 和 golang 的tar文件在同一目录下,执行

```bash
chmod +x k3s-init.sh
sudo ./k3s-init.sh
```