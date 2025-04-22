+++
date = '2024-11-21T14:42:19+08:00'
title = '玩转RISCV开发板01-烧录OpenEuler国产镜像'
categories = ["硬核技术"]
tags = ["RISC-V","OpenEuler","镜像烧录"]
+++

## 本次实验环境

* 宿主机:macbook air M2
* LiCheePi开发板，配置为16+128
* 一根USB-C线
* 一个拓展坞，键鼠，显示器
* 烧录镜像为openEuler/embedded_img/riscv64/lpi4a/

## 实验步骤

### 1.下载fastboot

这里使用homebrew快捷安装

```bash
brew install android-platform-tools
```

### 2.连接开发板与宿主机

先长按开发板上的BOOT按钮，再将USB线插入到宿主机上（拓展坞上），注意这里的先后次序。这一步是为了让开发板进入USB烧录模式。

进入终端，运行命令检查fastboot是否识别到了当前连接的开发板

```bash
fastboot devices
```

注意是devices不是device,结果如下

![](/img/riscv/fastboot.png)

这里我也没弄清楚为什么是一堆问号，不过只要能够检测到设备就好。（注意本教程目前只测试了Macos下的操作）

### 3.下载所需的镜像文件

来到[镜像下载网站](https://repo.openeuler.org/openEuler-24.03-LTS/embedded_img/riscv64/lpi4a/)选择以下三种文件进行下载：

* u-boot文件用于**引导加载程序**——系统会先启动一个小型的引导加载程序（SPL），它负责加载完整的 U-Boot 镜像
* base-boot.ext4.zst文件用于作为**系统的启动分区**——包括启动所需的文件，比如 Linux 内核、初始 RAM 磁盘（initramfs）、设备树（device tree）
* base-root.ext4.zst文件用于作为**根文件系统**——提供操作系统所需的文件

如果你安装过实打实的Linux系统（比如在windows下的双系统）的话你就会对这几个文件很熟悉。

值得注意的是，其使用Zst格式压缩，所以在真正使用的时候需要提前进行解压，如果忘了解压，直接操作你就会遇到这个问题

![](/img/riscv/fasterr.png)

哈哈，文件格式不对当然无法成功了。

故我们需要对zst格式结尾的文件进行解压

```bash
zstd -d openEuler-24.03-LTS-riscv64-lpi4a-base-boot.ext4.zst
zstd -d openEuler-24.03-LTS-riscv64-lpi4a-base-root.ext4.zst
```

### 4.进行烧录

首先进入上一步下载镜像所在的目录下

```bash
fastboot flash ram u-boot-with-spl-lpi4a-16g.bin
fastboot reboot
# wait five seconds
fastboot flash uboot u-boot-with-spl-lpi4a-16g.bin
fastboot flash boot openEuler-24.03-LTS-riscv64-lpi4a-base-boot.ext4
fastboot flash root openEuler-24.03-LTS-riscv64-lpi4a-base-root.ext4
```

最终如果成功你会得到一堆OKAY，这里放一张我的运行截图

![](/img/riscv/fastsuc.png)

### 5.验证

将开发板与宿主机的连接断开，将其连接到准备好的显示器上并准备好键鼠，等待其成功运行。
好的，成功发现一个神奇的问题，鼠标键盘也亮了，风扇也转起来了，结果显示器不亮？不亮？不亮？为什么！！！

![](/img/ys/药水挥拳.webp)

好，我先去google，来到官方文档烧录镜像的评论区发现一个遇到和我同样问题的朋友

![](/img/riscv/q1.png)

## 解决显示器不亮的问题

遇到这种问题，google并不能得到很相关的解决方案，这也是我在日常工作时觉得需要使用AI的场景之一。所以我得去询问ChatGPT让他针对于我这种问题给出**解决思路**。

即使镜像烧录是成功的，为了保险起见，我们先去利用官网的SHA256文件验证一下文件是否损坏。

```bash
sha256sum -c openEuler-24.03-LTS-riscv64-lpi4a-base-boot.ext4.zst.sha256sum
sha256sum -c openEuler-24.03-LTS-riscv64-lpi4a-base-root.ext4.zst.sha256sum
```

结果都显示OK,那么镜像文件是对的。

在检查ChatGPT给出的思路之前，我认为还是先来一遍对照实验，因为在烧录镜像之前自带的Debian是能够成功点亮显示器的，所以我们换一下镜像的版本来进行对照。

### 对照实验1-换镜像版本

换成兴趣小组的[镜像文件](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/preview/openEuler-23.03-V1-riscv64/lpi4a/)23.03再来一遍

命令与上面相同，其中`u-boot-with-spl-lpi4a-16g.bin`这个文件是可以复用的。

嘿，您猜怎么着，还是点不亮！

![](/img/ys/药水挥拳.webp)

那再换成[preview版本的23.09](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/preview/openEuler-23.09-V1-riscv64/lpi4a/)再来一遍，这次点亮了！并且给出了一个图形化的界面！正当我准备大展拳脚，以为自己终于找到了问题的所在时，又出差错了！

**没有wifi界面，无法连接wifi**，使用nmcli命令查看网络状态

```bash
nmcli general status
nmcli device status
```

显示WIFI是enabled,==WIFI-HW是missing的==，意味着WIFI硬件似乎没有被正确识别；只有本地环回地址连接成功了，其他都处于unmanaged状态。

我在其他的地方也找不到连接wifi的界面，所以这又是什么问题呢？（个人目前猜测还是镜像不稳定，因为Debian操作系统可以连接wifi）

最终，我去镜像文件中的devel部分找到[23.09的镜像](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/devel/20240122/v0.1/lpi4a/)进行测试，结果成功了！

### 成功解决

如上面所说，找到链接中的镜像后进行再次烧录,终于可以点亮屏幕（如果遇到键鼠没反应重启开发板即可），进入了一个黑色的命令行界面，接着我们要进行登录。

```bash
用户名:openeuler
密码:openEuler12#$
```

登录后我们首先需要连网，使用以下命令。

```bash
# 检查网络状态，去报WIFI-HW,WIFI是enabled
nmcli general status
# 查看端口和网络，确保有wlan端口
ip a
# 显示可用的wifi
nmcli dev wifi
# 连接具体wifi，其中SSID替换为wifi名，PASSWORD替换为密码
sudo nmcli dev wifi connect SSID password PASSWORD
# 连接成功后再检查STATE是否为connected
nmcli general status
```

至此，成功连接wifi，烧录镜像这一步骤到此结束。

## 后续问题

看到这里，还有惊喜！还没完！我重启了一次开发板之后，发现wifi又没了，瞬间尴尬在原地，到底是镜像和网卡你两谁的问题!?

![](/img/shu/开枪.webp)

又经历了两次重启，wifi又好了，所以，请看下面总结。

## 总结与反思

这里总结一下为LicheePI烧录OpenEuler镜像遇到的问题。

### 镜像or网卡问题

在还没烧录的时候，板子自带的Debian操作系统一切正常。接着烧录Openeuler for riscv镜像时出现了一系列的问题：

* 先遇到官方最新的**24.03LTS**版本的专门为LiCheePI准备的镜像烧录后无法打开显示器的问题。
* 在SIG兴趣小组提供的链接中，preview下的镜像23.03还是没有点亮显示器。
* 同样，preview下的23.09虽然点亮了显示器，但是无法识别网卡，连接不了wifi
* 于是去develp下的23.09镜像点亮了显示器也连接到了wifi，但是存在偶尔的抽风问题——重启后wifi又没了，得不断重启尝试。

所以，综上所述，是谁的问题？

我觉得八成原因在镜像，因为换了镜像后问题得到解决，不过如果细心的读者会发现我们对于23.03镜像并没有进行多次重启来检测是不是网卡的问题（其实我也重启了两次不过都没成功），所以还有两成原因在于WIFI网卡——看的这里你是否觉得这也太不“技术”了，一个技术博客怎么能够有点玄学成分了。

哈哈，你的批评是对的，在这里我确实不严谨了，没有程序员深挖精神了，私密马赛！

### 反思

最后的最后，我不禁想问自己一个问题：操作系统镜像和架构有关系吗？

那这个问题答案也很简单，一定是有关系的，操作系统镜像必须与CPU指令集兼容，这也是为什么操作系统镜像页面有专门为RISC-V所设计的镜像页面。

那么关于如何烧录OpenEuler for RISCV镜像到LiCheePi上这一主题就告一段落了，还是有点流水账了，anyway，博客永远是给自己看的。

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**