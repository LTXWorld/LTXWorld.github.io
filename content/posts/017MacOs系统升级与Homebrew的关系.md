+++
date = '2025-04-03T17:07:56+08:00'
title = 'DebugEP02——MacOs系统升级与Homebrew的关系'
categories = ["通用技术"]
tags = ["Homebrew","Macos","软件包管理"]
+++

## 引子

某天我想要在我的macos上使用homebrew安装[riscv编译工具链](https://github.com/riscv-software-src/homebrew-riscv/)为了交叉编译一些镜像文件。

```bash
brew tap riscv-software-src/riscv
brew install riscv-tools
```

即使其中遇到了一些网络问题，但都可以解决，直到遇到一个在我看来很神奇的问题，Homebrew与Xcode的版本有关，而Xcode的升级又必须依赖macos系统的升级。这就逼迫我这种不太喜欢升级系统的人不得不升级一下macos的系统。

所以这一顿操作结束后我就在思考，为什么？

## Homebrew简介

还记得大三第一次使用mac的时候，我觉得这不就是一个笔记本形态的ipad吗？下载软件也是从AppStore或者浏览器搜到的DMG文件，直到某天我在下载某个Github上的软件时看到有一段专门为macos写的命令`brew install xxx`，我才知道原来Homebrew才是macos下载东西的利器！

如果您没有使用macos，或许您没有听过Homebrew，不过简单地类比就是我们在linux下使用的apt，dnf，yum等包管理器。如果您了解linux的包管理器，apt和dnf在执行安装时都是直接安装二进制文件，而Homebrew采取了两种方式一种是安装二进制文件，一种是本地编译构建，因为**其本质是一个用Ruby编写的Git仓库项目。**

形象地描述一下Homebrew的执行原理——Homebrew这个软件安装助手其自身叫Brew,他要为我们安装软件时需要找到软件安装说明书即Formula软件安装脚本（用ruby编写），诸多的软件对应着诸多的安装脚本，我们使用Core这样一个Git仓库来保存这些脚本。当安装命令下达时，管家查找本地的Core仓库，寻找对应的脚本，找到后查看脚本中是否设置了对应的Bottle(二进制文件)链接，如果有则直接下载这个Bottle，如果没有，则利用安装脚本中指定的链接下载源码进行本地构建。

让我们根据[清华源](https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/)中给出的安装步骤再来看看Homebrew的执行原理。

```bash
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git"
export HOMEBREW_INSTALL_FROM_API=1

echo 'export HOMEBREW_API_DOMAIN="https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles/api"' >> ~/.zprofile
echo 'export HOMEBREW_BOTTLE_DOMAIN="https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles"' >> ~/.zprofile
export HOMEBREW_API_DOMAIN="https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles/api"
export HOMEBREW_BOTTLE_DOMAIN="https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles"

pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple

# 安装
git clone --depth=1 https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/install.git brew-install
/bin/bash brew-install/install.sh
rm -rf brew-install
# 加入环境变量
test -r ~/.zprofile && echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

在这里我们设置了`Homebrew_Brew`和`Homebrew_Core`的地址使用清华的镜像仓库，这样在使用Core仓库时就不会遇到网络问题；设置了`API_DOMAIN`和`BOTTLE_DOMAIN`的镜像地址，并且将`FROM_API`设置为1，意味着开启了API模式——接着上面的解释，之前Homebrew必须将Core仓库全部Clone到本地，每次更新时也会更新极为庞大的仓库，所以自2021年后，推出的API安装模式，不再需要Clone Core仓库，而是通过API访问`https://api.homebrew.sh/`找到指定的JSON文件，brew管家会解析这个JSON文件知道其Bottole在哪，用哪个架构，版本号是多少等各种信息。

最终下载好的二进制包默认在Cellar目录下，可执行文件会被设置一个软链接，让我们在终端可以直接运行此软件。最后补充一点，Homebrew对比apt,dnf来说更像是一个**用户的软件助手，而非系统管家，因为其不需要root权限。**

所以到这里对于Homebrew我们就有了一个大致的了解，后续我可能会对比一下这些包管理工具的异同，敬请期待。

## 为什么需要升级Xcode

首先解释为什么需要Xcode,前面说到，如果没有Bottle,Homebrew会进行本地构建，这里更关键的是，Homebrew本身不处理编译过程，而是依赖于系统工具链，对于macos来说就是Xcode的clang和make，通过`xcode-select --install`安装。

我的场景是要安装riscv的编译工具链，因为使用了`brew tap`，添加了第三方的formula仓库，会去Github上clone这个仓库，所以其一定会触发本地编译构建过程。

其次为什么要升级，执行`brew doctor`这个诊断指令，系统会提示你版本过低。

```bash
Error: Your Command Line Tools are too outdated.
Please update them via Software Update or `xcode-select --install`.
```

并且Xcode提供的CLT工具链会调用macos的系统库，所以我们必须要升级macos的系统。

## 总结Homebrew操作

写到这里，我对Homebrew的理解已经比较清晰了，所以同时来总结一下其可能的操作流程。

```bash
# 查看基本信息
brew info <appName>
# 查看依赖项
brew deps <appName>
brew deps --tree <appName>
# 
brew install <appName>
# 如果需要tap指定第三方仓库,然后再安装
brew tap
# 列出安装详细过程
brew install <appName> --verbose
# 列出已安装的
brew list
<appName> --version
# 检查配置操作和潜在问题
brew doctor
# 更新brew本体和formula
brew update
# 升级所有安装软件
brew upgrade
```

举例说明,这个RiscV工具链是本地构建的。

```bash
brew info riscv-tools
==> riscv-software-src/riscv/riscv-tools: stable 0.2
RISC-V toolchain including gcc (with newlib), simulator (spike), and pk
http://riscv.org
Installed
/opt/homebrew/Cellar/riscv-tools/0.2 (5 files, 54.5KB) *
  Built from source on 2025-04-03 at 18:05:56
From: https://github.com/riscv-software-src/homebrew-riscv/blob/HEAD/riscv-tools.rb
==> Dependencies
Required: riscv-gnu-toolchain ✔, riscv-isa-sim ✔, riscv-pk ✔
(base) lutao@MagicTech Downloads % brew deps riscv-tools
dtc
gmp
isl
libmpc
lz4
mpfr
riscv-software-src/riscv/riscv-gnu-toolchain
riscv-software-src/riscv/riscv-isa-sim
riscv-software-src/riscv/riscv-pk
xz
zstd
```

这个Go语言来自于Bottle

```bash
(base) lutao@MagicTech Downloads % brew info go
==> go: stable 1.24.2 (bottled), HEAD
Open source programming language to build simple/reliable/efficient software
https://go.dev/
Installed
/opt/homebrew/Cellar/go/1.24.2 (14,109 files, 283.8MB) *
  Poured from bottle using the formulae.brew.sh API on 2025-04-03 at 17:31:38
From: https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git/Formula/g/go.rb
License: BSD-3-Clause
```

## 启发

对于我们日后镜像的迁移构建，其带来了一定的启发

- 把每一步构建流程脚本化、配置化
- 提高可复现性，适配 CI/CD
- 缓存和复用已构建的镜像或软件包
- 构建前先 resolve 所有依赖
- 类似 bottle 仓库 + fallback 编译方式

那么本篇文章到这里就结束了，**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**