+++
date = '2025-05-15T13:43:20+08:00'
title = 'K3sEP10——迁移k3s至riscv64开发板的过程记录'
categories = ["云计算"]
tags = ["K3s","RiscV"]
+++

## 引子

在前面的一系列文章中,我们知道了要想在 riscv 开发板上适配 k3s,我们要做以下两件事

- 系统级别的重要镜像进行适配并可以成功拉取到开发板
- 执行 make 或者 download,build,package-cli 脚本

对于第一件事,已经有人做过了,我们只需要使用他做好的镜像,并将其推送到我们的私有仓库.

![support](./img/riscv/support.png)

对于第二件事,在操作过程中我们可能会遇到大大小小的问题,其实在之前的文章操作中我们都遇到过,这里汇总一下.

## 构建过程

由于 make 过程会执行 dapper 的容器构建,会涉及到下载,由于众所周知的网络问题(我不可能给开发板上代理吧),所以我们放弃 make 过程.

直接采用执行三个脚本的方法,在此过程中我们需要修改一些部分的代码.

### 避免Golang版本检测

由于在脚本执行过程中,会去检测 当前代码中 k8s 的 Golang 的版本,具体代码在 `/scripts/validate` 如下:

```bash
echo Running: go version
if ! go version | grep -s "go version ${VERSION_GOLANG} "; then
  echo "Unexpected $(go version) - Kubernetes ${VERSION_K8S} should be built with go version ${VERSION_GOLANG}"
  exit 1
fi
```

这个 VERSION_GOLANG 来自于 `/scripts/version.sh`

```bash
DEPENDENCIES_URL="https://raw.githubusercontent.com/kubernetes/kubernetes/${VERSION_K8S}/build/dependencies.yaml"
VERSION_GOLANG="go"$(curl -sL "${DEPENDENCIES_URL}" | yq e '.dependencies[] | select(.name == "golang: upstream version")
```

故它会去使用 curl 命令查找这个 yaml 文件,接着 yq 读取其中 golang 的字段,具体内容如下:

```yaml
# Golang
  - name: "golang: upstream version"
    version: 1.22.8
    refPaths:
    - path: .go-version
    - path: build/build-image/cross/VERSION
    - path: staging/publishing/rules.yaml
      match: 'default-go-version\: \d+.\d+(alpha|beta|rc)?\.?(\d+)?'
    - path: test/images/Makefile
      match: GOLANG_VERSION=\d+.\d+(alpha|beta|rc)?\.?\d+
```

而这个检查 golang 版本的行为会出现在我们运行 k3s 的过程中,源码中 server 和 agent 的启动第一步就是 `cmds.MustValidateGolang()`

但是,由于众所周知的网络问题, curl 不动,😅;如果你不管这个问题就会出现以下现象:

```bash
May 14 18:16:01 revyos-lpi4a k3s[1447]: time="2025-05-14T18:16:01+08:00" level=fatal msg="Failed to validate golang version: incorrect golang build version - kubernetes v1.31.2 should be built with go, runtime version is go1.23.6"
```

可以发现,`should built with go`这是错误的,至少后面也应该是 goxxx,而不是 go——证明其没有读取到 yaml 文件中的内容.

这也就导致运行 k3s 时第一步的检查版本代码过不去,即下方这里的代码: `pkg/cli/agent/agent.go`

```go
func ValidateGolang() error {
	k8sVersion, _, _ := strings.Cut(version.Version, "+")
	if version.UpstreamGolang == "" {
		return fmt.Errorf("kubernetes golang build version not set - see 'golang: upstream version' in https://github.com/kubernetes/kubernetes/blob/%s/build/dependencies.yaml", k8sVersion)
	}
	if v, _, _ := strings.Cut(runtime.Version(), " "); version.UpstreamGolang != v {
		return fmt.Errorf("incorrect golang build version - kubernetes %s should be built with %s, runtime version is %s", k8sVersion, version.UpstreamGolang, v)
	}
	return nil
}
```

所以我们最后干了什么:

- 将 validate 中关于检验 Golang 版本的代码注释掉
- 将 server 与 agent 中的 `cmds.MustValidateGolang()`注释掉
- 在 yaml 文件中找到适合的 golang 版本作为我们开发板上要用的版本,见上一篇文章环境配置.

### 避免下载过程过慢

在执行 download 过程中,其实并不需要在开发板上执行,这一点我们在[这篇文章](https://www.bfsmlt.top/posts/023k3sep06%E4%BB%8Eissues%E4%B8%8A%E5%BE%97%E5%88%B0%E7%9A%84%E5%8F%AF%E8%83%BD%E5%B0%9D%E8%AF%95/#download%E9%81%87%E5%88%B0%E7%BD%91%E7%BB%9C%E9%97%AE%E9%A2%98)中提过,具体做法如下:

```bash
# 在我们的主机上执行
export GOARCH=riscv64
export ARCH=riscv64
export OS=linux

./scripts/download
# 然后拷贝build目录到开发板的k3s源码目录中
scp build/ root@ip:/root/k3s
```

### 开发板 airgap 安装

在开发板上执行 build 与 package-cli 脚本,build过程中可能会遇到网络问题,速度较慢,但是只要我们的环境,工具都正确,不会出现其他问题.

执行完两个脚本后会在 `/dist/artifacts`目录下产生一个 k3s-riscv 可执行二进制文件,将作为我们后续安装的文件.

接着我们进行 [airgap](https://docs.k3s.io/zh/installation/airgap) 安装 server 节点,利用上一篇文章中的安装脚本即可完成安装.

安装后检查是否成功:

```bash
sudo systemctl status k3s
sudo kubectl get nodes
```

看到自己的节点即为成功.

### 手动拉取镜像

在 2025-05-14 实验版本中,系统镜像并没有自动拉取,这点我们还在尝试,因为官网还有一种为其设置配置文件的方式 `/etc/rancher/k3s/config.yaml`

所以我们使用 `sudo kubeclt get pods -A` 来查看这些系统镜像,并用`sudo kubectl describe pod <podName> -n kube-system` 来查看其错误原因

错误原因大部分都来自于镜像拉取失败,因为它还在拉取默认的 rancher 镜像.
所以我们需要修改 registries.yaml 文件,为其添加 rewrite,使其将默认镜像改为我们私有仓库中的镜像.

```yaml
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

然后需要手动拉取这些镜像,利用 `sudo crictl pull` 命令拉取.
最终结果如下:

```bash
sudo kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS      AGE
kube-system   coredns-56f6fc8fd7-28mn7                  1/1     Running     1 (18h ago)   19h
kube-system   helm-install-traefik-crd-98v8n            0/1     Completed   0             19h
kube-system   helm-install-traefik-j7czp                0/1     Completed   0             19h
kube-system   local-path-provisioner-5cf85fd84d-hznms   1/1     Running     0             19h
kube-system   metrics-server-5985cbc9d7-v7dch           1/1     Running     0             19h
kube-system   svclb-traefik-699324d1-48lvn              2/2     Running     0             93m
kube-system   traefik-57b79cf995-xnv79                  1/1     Running     0             93m
```

这些系统镜像都已经在成功运行.

### 测试pod

在主机上构建一个小的镜像推送到私有仓库中去

```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func handler(w http.ResponseWriter, r *http.Request) {
    message := os.Getenv("MESSAGE")
    if message == "" {
        message = "Hello from RISC-V!"
    }
    fmt.Fprintf(w, "%s\n", message)
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("Listening on :7080")
    http.ListenAndServe(":7080", nil)
}
```

```dockerfile
FROM golang:1.22.12-alpine3.20 AS builder
WORKDIR /app
COPY . .
RUN go build -o hello-k8s main.go

FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/hello-k8s .
EXPOSE 7080
ENTRYPOINT ["./hello-k8s"]
```

构建并推送,注意设置 platform 与 network

```bash
docker build --platform=linux/riscv64 --network host -t jimlt.bfsmlt.top/hello-kubernetes:1.0.1 .
docker push jimlt.bfsmlt.top/hello-kubernetes:1.0.1
```

这里看着 push 的结果可以思考一下为什么 push 的时候显示 traefik?

```bash
The push refers to repository [jimlt.bfsmlt.top/hello-kubernetes]
f8b25a4c3a38: Pushed
ecbb5a20f6d5: Pushed
7df33f7ad8be: Mounted from traefik
4f4fb700ef54: Layer already exists
1.0.1: digest: sha256:78107dc8255d5b7eb09031d1c26fee9b10ee896237ac8b8acc1914e83b704325 size: 857
```

- 因为私有仓库中的 traefik 镜像已经有了 7d 这个镜像层(Docker 镜像的基本概念,镜像由层组成)
- 故这里复用了这个镜像层,就不用再重新构建这层了

在开发板上写一个 hello-deploy.yaml 进行部署测试

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  type: ClusterIP
  ports:
    - port: 7080
      targetPort: 7080
  selector:
    app: hello
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:  # 注意这里在 spec 下，不是在 selector 下
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello-kubernetes
          image: jimlt.bfsmlt.top/hello-kubernetes:1.0.0
          env:
            - name: MESSAGE
              value: "Hello RISC-V!"
          ports:
            - containerPort: 7080
```

运行此 deployment 并执行测试

```bash
sudo kubectl apply -f hello-deploy.yaml
sudo kubectl port-forward svc/hello 7080:7080
curl http://localhost:7080
# 输出结果
Hello RISC-V!
```

## 总结

至此,我们的移植过程暂时告一段落了,可以运行起一个简单的小应用,接下来我们会将另一个开发板作为 agent 加入其中进行集群测试.

不过其中对源码的修改方式(避免检测)也是一种无奈的尝试,后续看看有没有什么其他的好的方式;还有手动拉取系统镜像这一步也不应该这样做,再看吧.

最后问自己一句:问题解决了吗?**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**

![2](/img/shu/躺平.webp)