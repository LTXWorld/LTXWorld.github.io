+++
date = '2025-05-16T15:23:10+08:00'
title = '兴趣八股之计算机网络EP03——Ping命令'
categories = ["计算机网络"]
tags = ["Ping","八股","IP"]
+++

## 引子

在与开发板打交道的过程中,网络成为了出现问题最多的地方,之前我们谈过 DNS 的事情,今天我们来说说 Ping 这个命令,每次网络出问题的时候最喜欢的命令就是 

```bash
ping baidu.com # 最喜欢 ping 前司了
ping 8.8.8.8
```

那 ping 命令到底是用来干什么的呢?其中的原理是什么呢?接下来让我们一起探索一下.

## 什么是 ping

Ping 即是一个动词又是一个名词,名词在于他本身就是一个应用,动词在于我们常常使用 ping 命令.

甚至,我们平时打游戏的时候常说的就是这个 ping,当你的 ping 值很高时,那代表你可能有些卡了!

### 原理

当我们在主机上使用 `ping IP/DomainName` 时

1. 我们的主机就会向远程的服务器发送多个 **ICMP echo requestes**

- 这是 ICMP 协议中专门用于测试网络连通性的一种报文;
- 其对应的就是 **ICMP echo reply**,众所周知,ICMP 作用于网络层,IP所在的层.
- 这个报文会带着一个发送的时间戳和序号
- 其不依赖于 TCP/UDP,而是直接封装在 IP 包中,所以自然是一个不可靠传输.

2. 如果目标主机可达,返回响应
3. Ping 工具靠着应答时间与发送时间计算时间差即 RTT,并统计包数量,计算丢包率,同时还提供 TTL 生存时间值(可以经过多少个路由器);如下所示

```bash
ping baidu.com
PING baidu.com (110.242.68.66): 56 data bytes
64 bytes from 110.242.68.66: icmp_seq=0 ttl=47 time=68.472 ms
64 bytes from 110.242.68.66: icmp_seq=1 ttl=47 time=47.705 ms
64 bytes from 110.242.68.66: icmp_seq=2 ttl=47 time=30.750 ms
64 bytes from 110.242.68.66: icmp_seq=3 ttl=47 time=37.783 ms
64 bytes from 110.242.68.66: icmp_seq=4 ttl=47 time=43.311 ms
64 bytes from 110.242.68.66: icmp_seq=5 ttl=47 time=33.979 ms
^C
--- baidu.com ping statistics ---
6 packets transmitted, 6 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 30.750/43.667/68.472/12.430 ms
```

关于数据包传递的细节,我们后续在其他文章中更新,这会涉及到网络层的内容.

### 用途

知道了 Ping 的工作原理,我们来看看他有什么用途:

1. 最常见的就是验证远程的服务器,网站,网络设备能不能通过我们的网络访问
2. 从结果可以看到,还可以测量数据包的 RTT(time字段) 和丢包率来评估网络的延迟情况
   1. 丢包率受多个因素影响,可能是防火墙,可能是网络拥塞
3. 还可以测试本地环回地址以此来验证本地的 TCP/IP 栈是否正常

所以回到上面游戏里的 ping 值,当 ping 值过高,就表示着我们与当前连接的远程服务器之间的网络并不畅通,数据包的传输比较慢,或者丢包严重,自然就造成游戏操作变卡.

### traceroute

看完了上面 ping 的作用,可以顺便思考一下,Ping 不能看到什么?

没错,ping 看不到数据包在网络中的路径——从哪里到哪里的.
所以我们需要另外一个工具来查看路由中的每个跳转,以此来跟踪整条路径,判断出在哪里出现了丢包,超时等情况.

traceroute 的工作过程如下:

- 发送第一个 TTL = 1 的数据包 → 第一个路由器返回 “TTL exceeded” → 记录第 1 跳 IP 和 RTT
- 然后发送 TTL = 2 的数据包 → 第二个路由器返回 → 记录第 2 跳
- 一直增加 TTL，直到目标主机响应，或者达到最大 TTL

```bash
traceroute to google.com (172.217.31.142), 64 hops max, 40 byte packets
 1  * * *
 2  * * *
 3  * * *
```

从这例子来看,很可惜,应该是 google 的防火墙屏蔽了我的请求.但这并不意味着 google 不能访问.

那再试试我的路由器.

```bash
traceroute 192.168.1.1
traceroute to 192.168.1.1 (192.168.1.1), 64 hops max, 40 byte packets
 1  192.168.1.1 (192.168.1.1)  30.071 ms  2.634 ms  2.095 ms
```

很明显,从我的主机到路由器就只需要一跳.

所以,我们日后在排查的时候可以将 ping 和 traceroute 结合起来,前者用于检测是否可以连通,后者来检查如果有问题哪里出了问题.

### mtr

这是一个更强的 traceroute,mtr 会实时显示每跳的 RTT 和丢包情况，比 traceroute 更直观.推荐使用.

![1](/img/bg/mtr.png)

可以看到,分为前几跳,中间几跳和最后.

- 前几跳可能在局域网中
- 中间几跳是具体的 IP 地址,这些是电信的骨干网,特别是最后一个是电信的网络出海口.
- 有些运营商路由器默认不回应 mtr 的 ICMP/UDP 包，这不代表一定有问题

### Ping spoofing

> where the main goal is to overwhelm the victim's server by sending it an extremely large number of echo request packets within a short period of time.

具体而言,攻击者可以伪造 ping 命令的 ICMP 包,将一个被害者的 IP 作为 源 IP,目标主机随意;
之后对一批目标主机发起 ping 命令,那么就会造成数不胜数的 ICMP reply 返回给被害者,形成反射攻击,是 DDoS 攻击的一种,称为 DDos放大.

```bash
hping3 -1 --spoof 10.0.0.2 8.8.8.8 # 被害者为10.0.0.2
```

如何避免呢?这里仅仅列出几个常见做法,没有深究具体过程.

- 过去的方法是采用防火墙即 iptables 限制 ping 的速率和来源
- 现在常用的有入侵检测系统,会检查异常的 ICMP 活动
- 对源地址进行验证(反向路径过滤)

## 实际应用

在实际的开发板操作过程中,有时遇到 ping 不通的情况,但实际情况就是自己的网卡,网络不太行.(因为我的随身wifi不太好)

在网络正常的情况下,我 `ping 8.8.8.8, baidu.com` 都是成功的.
这里就要多嘴一句,你认为 ping 命令与 DNS 有关吗?一个网络层,一个应用层,好像...

其实是可能有关的,例如 `ping google.com` 我们需要用到 DNS 来解析此域名为 IP 地址.

在这个命令中会先触发一次 DNS 得到 IP 地址,然后通过 socket 接口传给内核网络栈,最终传给网络层去封装,再层层递进...

## 总结

总结一下,这篇文章还是比较简单的,因为 ping 命令可能从小时候的微机课都或多或少地接触过了,一个人人都能上手的检测网络连通性的好办法.

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**

## 引用

- https://www.techtarget.com/searchnetworking/definition/ping
- ChatGPT