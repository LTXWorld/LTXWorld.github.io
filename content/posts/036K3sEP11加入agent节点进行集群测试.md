+++
date = '2025-05-16T14:08:23+08:00'
title = 'K3sEP11——加入agent节点进行集群测试'
categories = ["云计算"]
tags = ["K3s","RiscV","Redis"]
+++

## 引子

继上一篇文章中我们成功迁移了 k3s 到 riscv64 上之后,我们使用另外一个 riscv 开发板作为 agent 节点进行集群化测试.

## 环境初始化

仍然按照 EP09 中的步骤进行环境的初始化.

执行到安装时,请按照下面的步骤进行,因为 agent 与 server 的步骤略有不同.

## 部署agent

首先,我们将 k3s 的可执行文件放入 agent 中.

```bash
sudo cp /home/debian/k3s /usr/local/bin/k3s
sudo chmod +x /usr/local/bin/k3s
```

接着,下载安装脚本并写好配置文件(这次我们采用预先配置的方法进行安装)

```bash
curl -sfL https://get.k3s.io -o k3s-install.sh
# 编写config
sudo vi /etc/rancher/k3s/config.yaml
# 内容如下:
server: https://<SERVER_IP>:6443
token: <YOUR_TOKEN>
node-name: agent197
```

这里的 SERVER_IP 就是我们的 server 节点的 IP 地址; token 需要在 server 节点中获取

```bash
# 来到 server 节点
cat /var/lib/rancher/k3s/server/token
# 复制得到的结果
```

接着,继续配置 agent 上的镜像拉取规则,即 registries.yaml;这里我们设置与 server 相同的配置

```bash
sudo vi /etc/rancher/k3s/registries.yaml
# 填充内容如下
mirrors:
  docker.io:
    endpoint:
      - "https://jimlt.bfsmlt.top"
    # 这里根据上
    rewrite:
      "^library/pause$": "pause"
      "^rancher/mirrored-library-traefik$": "traefik"
      "^rancher/mirrored-metrics-server$": "metrics-server"
      "^rancher/klipper-helm$": "klipper-helm"
      "^library/klipper-lb$": "klipper-lb"
      "^rancher/local-path-provisioner$": "local-path-provisioner"
      "^rancher/mirrored-coredns-coredns$": "coredns/coredns"
      "^busybox$": "riscv64/busybox"

# 注意,因为我们采用了 TLS 的私有仓库,所以这里需要添加 configs
configs:
  "jimlt.bfsmlt.top": # 你的私有仓库地址
    auth:
      username: xxx
      password: yyyyyyyy
```

最后,我们使用命令部署 agent,跳过下载,并设置为 agent 节点

```bash
INSTALL_K3S_SKIP_DOWNLOAD="true" INSTALL_K3S_EXEC="agent" bash -x k3s-install.sh
```

至此,我们完成了 agent 部署,用以下步骤来验证其成功运行.

```bash
# 在 agent 上
sudo systemctl status k3s-agent
# 在 server 中
sudo kubectl get nodes
```

得到结果如下:

```bash
NAME           STATUS   ROLES                  AGE    VERSION
agent197       Ready    <none>                 175m   v1.31.2+k3s-4c017da7
revyos-lpi4a   Ready    control-plane,master   41h    v1.31.2+k3s-4c017da7
```

值得注意的是, agent 只是工作节点,我们的那些系统组件例如 coredns 只会在 server 节点执行;除了 pause 镜像以外,这是一个最最最基础的镜像,无论如何都需要.

同样的, agent 上无法进行 `kubectl` 操作,因为 kubectl 需要与 API Server 通信,而 API Server 默认运行在 Server 节点上.

最后我们再来查看一下集群的信息.

```bash
sudo kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

## 测试 redis 应用

在 Server 上写好两个 yaml 文件,一个作为服务端,一个座位客户端.

服务端,redisTest-deployment.yaml:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis 
  template:
    metadata:
      labels:
        app: redis # 设置 pod 标签为 app: redis
    spec:
      containers:
      - name: redis
        image: jimlt.bfsmlt.top/redis:v7.2.4
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis # 将流量转发给 app: redis 的 pod
  ports:
  - port: 6379
    targetPort: 6379
    protocol: TCP
  type: ClusterIP
```

客户端,redisTest-client.yaml:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: redis-client
spec:
  containers:
  - name: redis-client
    image: jimlt.bfsmlt.top/redis:v7.2.4 # 自建的 RISC-V 客户端镜像
    command: ["sleep", "3600"]
```

在服务端执行`kubectl apply -f redis-deployment.yaml`和`kubectl apply -f redis-client.yaml`

可以查看各自的 pod 信息,查看 service 信息与实际转发的信息

```bash
# pod 信息
sudo kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS      AGE     IP           NODE           NOMINATED NODE   READINESS GATES
hello-7ff6b9f9bc-2kfff   1/1     Running   4 (12m ago)   21h     10.42.0.40   revyos-lpi4a   <none>           <none>
redis-7c667cb775-d72lw   1/1     Running   1 (12m ago)   109m    10.42.0.42   revyos-lpi4a   <none>           <none>
redis-client             1/1     Running   0             9m30s   10.42.1.9    agent197       <none>           <none>
# service 信息
sudo kubectl get svc redis-service -o wide
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
redis-service   ClusterIP   10.43.184.199   <none>        6379/TCP   5h59m   app=redis
# 实际转发的信息
sudo kubectl get endpoints redis-service
NAME            ENDPOINTS         AGE
redis-service   10.42.0.42:6379   6h
```

查看 redis-client 的构建过程,可以发现其被 API_Server 调度到了 agent197 上

```bash
sudo kubectl describe pod redis-client
# 
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  70s   default-scheduler  Successfully assigned default/redis-client to agent197
  Normal  Pulled     69s   kubelet            Container image "jimlt.bfsmlt.top/redis:v7.2.4" already present on machine
  Normal  Created    69s   kubelet            Created container redis-client
  Normal  Started    68s   kubelet            Started container redis-client
```

进入客户端 Pod 并连接 Redis,指定连接的 redis 服务器地址为 redis-service,这在 yaml 文件中指明了,会自动解析成 Redis Pod 的 ClusterIP 地址

```bash
sudo kubectl exec -it redis-client -- sh
# 在容器内执行
redis-cli -h redis-service
```

执行一些命令进行测试,因为我的 redis 配置文件中配置了密码,所以执行 `auth {password}`时视自己情况而定

```bash
redis-service:6379> get name
(nil)
redis-service:6379> set name jimlt
OK
redis-service:6379> get name
"jimlt"
```

至此,这个简单的 redis 测试应用就部署成功了.

## 构建多架构镜像(可选)

如果我们后期将其他架构的机器加入到集群里面,我们在拉取镜像时可能遇到架构不兼容问题,所以我们需要构建一个多架构的 List 在私有仓库中.

在 macos 上执行下面的命令,其中 Dockerfile 来自于[这篇文章](http://bfsmlt.top/posts/016k3sep03%E7%A7%BB%E6%A4%8D%E9%95%9C%E5%83%8F%E5%88%B0riscv%E5%BC%80%E5%8F%91%E6%9D%BF/#%E4%BC%98%E5%8C%96%E9%95%9C%E5%83%8F).

```bash
docker build --platform linux/amd64,linux/arm64,linux/riscv64 --network host -f Dockerfile.redis -t jimlt.bfsmlt.top/redis:multiarch .
```

但是却遇到 riscv64 有关的错误,这在我们最初构建时并没有发生,目前暂时没有找到好的处理方法.

```bash
5.934     CC adlist.o
5.934 cc: error: unrecognized argument in option '-mabi=lp64d'
5.934 cc: note: valid arguments to '-mabi=' are: ilp32 lp64; did you mean 'lp64'?
5.934 make[1]: *** [Makefile:436: adlist.o] Error 1
5.934 make[1]: Leaving directory '/redis/src'
5.934 make: *** [Makefile:6: all] Error 2
```

## 总结

k3s 已经成功地在一个由 riscv 开发板所组成的集群中运行起来了,可以创建,调度 Pod,可以做到 Pod 之间的通信.

在这里,需要对 [Antony Chazapis](https://github.com/CARV-ICS-FORTH/kubernetes-riscv64) 对 k3s for riscv 所作出的贡献致以最高的敬意!

当然,这只是长征路上的第一步,我们才刚刚起步而已,后续围绕 k3s 进行的分布式框架编写,分布式应用的依赖分析以及最后要形成的一键部署工具都还是任重道远啊.

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**