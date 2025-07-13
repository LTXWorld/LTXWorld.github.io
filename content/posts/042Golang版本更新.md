+++
date = '2025-07-11T11:40:27+08:00'
title = '042Golang版本更新'
categories = ["通用技术"]
tags = ["Golang","版本管理"]
+++

## 引子

在使用 Go 语言开发过程中，我发现经常会遇到想使用的项目的 Go 版本与当前本机的 Go 版本不一致的情况，通常是本地的版本较低。

所以每次都需要去手动更新版本，而手动更新的过程是比较繁琐的，需要下载新版本并替换旧版本（听起来也没什么是吧）但是 Go 版本的更新还算是比较频繁的，特别是各种小版本。

所以我寻思着一定有什么自动化的更新或者可以管理本地的 Go 版本的工具吧，于是在浏览 Reddit 的过程中找到了 **gvm** 和 **g** 这两个 Go 版本工具。

在进入 gvm 之前，我想了想，自己的 Go 是怎么安装的呢？使用命令 `go env | grep GOROOT` 查看路径发现我是通过 Homebrew 安装的。

所以执行 `brew upgrade go`应该是可行的，但是怎么说呢，速度感人啊,一共花了 41s

故之后的 gvm 测试我决定拿到开发板上进行测试，之前我们在开发板环境搭建中不就遇到了 Golang 版本的切换问题吗？🧐

## gvm

听名字有点像 Java 中的 jvm 不是吗？Anyway,我使用开发板进行这个测试。

安装

```bash
# 安装bison
sudo apt-get install bison
# 安装gvm
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)
Cloning from https://github.com/moovweb/gvm.git to /home/debian/.gvm
Created profile for existing install of Go at /usr/local/go
Installed GVM v1.0.22

Please restart your terminal session or to get started right away run
 `source /home/debian/.gvm/scripts/gvm`
```

测试命令

```bash
gvm list all
# 安装1.24.5
gvm install 1.24.5
```

额，发现其下载速度感人，最后下载还显示错误`ERROR: Unrecognized Go version`

## g

安装时就遇到问题，我发现其在 riscv64 架构上并不适配。

```bash
curl -sSL https://raw.githubusercontent.com/voidint/g/master/install.sh | bash
[1/3] Downloading https://github.com/voidint/g/releases/download/v1.8.0/g1.8.0.linux-.tar.gz
```

可以看到 `linux-` 后面缺了一块，在其 release 中也发现暂未支持 RiscV 架构，但是用 Go 写的，那自己编译试试呗。

## 手动编译适配

其实对于 Golang 项目，其对于 RiscV 架构的适配是比较简单的，一方面因为 RiscV 架构比较新，没人用啊，哈哈；另一方面 Go 本身是适配 RiscV 架构的，所以这并不是一件难事。

先手动编译一下

```bash
git clone https://github.com/voidint/g.git
cd g
GOOS=linux GOARCH=riscv64 go build
```

这会编译出一个 g 二进制文件，接下来将其移动到 bin 中

```bash
mkdir -p ~/.g/bin
mv g ~/.g/bin/g
export PATH="$HOME/.g/bin:$PATH"
source ~/.bashrc
```

测试使用，注意使用之前先卸载掉之前已经安装好的 Go,或者修改 bashrc 中关于 Go 的路径设置。

```bash
g ls-remote
g install 1.24.5
```

Oh,居然没有遇到架构问题？damn，成功了，看来这两个 Go 系统管理系统在安装逻辑这里有区别哦。

```bash
g install 1.24.5
Downloading 100% [===============] (76/76 MB, 9.0 MB/s)
Computing checksum with SHA256
Checksums matched
Now using go1.24.5 linux/riscv64
```

可以发现 go 文件夹安装到了 `~/.g` 路径下,所以我们来修改 bashrc 中的路径。

```bash
# g: Go 版本管理器
export PATH="$HOME/.g/bin:$PATH"

# 设置 g 当前版本的 GOROOT 和 PATH
export GOROOT="$HOME/.g/go"
export PATH="$GOROOT/bin:$PATH"

# 设置 GOPATH（你的 go mod 或 go install 会安装到这里）
export GOPATH="$HOME/go"
export PATH="$GOPATH/bin:$PATH"
```

至此 g 可以在开发板上顺利切换版本。

```bash
# 列出当前已安装的版本
g ls
* 1.24.2
  1.24.5
# 使用某个已经安装的具体版本 
g use 1.24.5
Now using go1.24.5 linux/riscv64
```

## 不同点

### g install 流程

首先来到 g 的源码中关于 install 的部分，大致梳理一遍下载流程。

- 获取要下载的版本名称
- 构建 NewCollector
  - 根据 url 选择构建何种的 Collector
  - 将 url 预解析为 pURL,内容更加丰富，带有 Host,Path 等字段
  - 对 url 发送 Get 请求，使用 goquery 读取响应的 HTML
- 获取所有的版本信息保存在 items 中
- 根据版本名称，os，arch 信息找到 items 中对应的信息
- 拼凑出将要安装到 g 的路径,例如 `/Users/lutao/.g/versions/1.22.5`，并检查其是否已存在
- 从 items中找到符合 ArchiveKind, os, arch 的包文件(可能不止一个)
- checkSum
- 拼接 fileName 作为要从远程下载的文件名称 `/Users/lutao/.g/downloads/go1.22.5.darwin-arm64.tar.gz`
- 下载`return httppkg.Download(pkg.URL, dst, os.O_CREATE|os.O_WRONLY, 0644, true)`
- 解压下载的文件
- 切换版本

![1](/img/goproject/g.png)

最终在`~/.g`的目录下会有

- downloads：不同版本的 go 源文件
- versions:解压源文件得到的文件
- go：软链接，指向versions中的某个版本的文件

### gvm install 流程

与 g 不同的是，gvm 是以一个完全的 shell 脚本运行的，但是底层逻辑差不多都是从官方仓库拉取源码进行安装，设置路径，文件目录等。

在`download_binary`的部分对架构和 os 进行了判断，显然其中没有 RiscV 架构，所以我们之前安装失败。

## 向g项目学习

因为自己做的毕设可能也要开发命令行工具，所以 g 项目对我而言吸引力还是蛮大的。

我顺便看了看 ls,ls_remote 等功能是如何实现的，下面简单谈谈吧。

### ls

本质是`os.ReadDir()`方法的运用，读出的文件夹格式如下

- parent
- name
- typ
- info

接着利用 name 构造 version 结构体：`namej,sv,pkgs`,并按升序排序。
> Version represents a Go language distribution version.

调用 render，根据传来的版本信息打印出结果。值得学习的是，`color.New().Fprintf()`这样的着色式打印信息。

## 做出适配贡献

着眼于 Makefile,打开后发现其对于 linux 的构建规则中没有 riscv64，所以我们进行添加

```bash
build-linux: build-linux-386 build-linux-amd64 build-linux-arm build-linux-arm64 build-linux-s390x build-linux-riscv64

build-linux-riscv64:
	GOOS=linux GOARCH=riscv64 $(GO) build $(GO_FLAGS) -o  bin/linux-riscv64/g
```

同时，看到其安装方式是通过安装`install.sh`脚本实现，所以也要修改这个脚本。

原本脚本中是这样的，没有 riscv 架构，所以会出现最开始那种情况 `linux-` 缺了一块关于架构的。

```sh
function get_arch() {
    a=$(uname -m)
    case ${a} in
    "x86_64" | "amd64")
        echo "amd64"
        ;;
    "i386" | "i486" | "i586")
        echo "386"
        ;;
    "aarch64" | "arm64")
        echo "arm64"
        ;;
    "armv6l" | "armv7l")
        echo "arm"
        ;;
    "s390x")
        echo "s390x"
        ;;
    *)
        echo ${NIL}
        ;;
    esac
}
```

添加关于 riscv64 的字段即可。

最后我们将其汇总为一个 Pull Request 提交到了仓库中。

## 总结

Golang 的版本更新需要一个管理工具，反观 Java，在学习的时候好像很少有人提到其版本的更新，听到最多的是 Java 使用的版本要不然是 Java8 要不然就是 17，选一个很稳定的一直用。

Java 的版本工具最常用的是 SDKMAN（AI 回答）。

最终我们选择了 g 这个命令行项目作为我在 RiscV 开发板上的 Go 版本管理工具，并且这个项目有很多关于写命令行操作中值得学习的地方，最后我们提出了相应的 PR 🫡

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**