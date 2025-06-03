+++
date = '2025-06-03T09:27:10+08:00'
title = 'K3sEP12——遇到的Traedfik问题'
categories = ["云计算"]
tags = ["K3s","RiscV","Traefik"]
+++

## 引子

在进行 k3s 的适配过程中，Traefik 作为系统组件镜像之一，我们最终使用了大佬构建好的镜像作为私有镜像，并没有进行深究。

恰好在昨天的一个集群测试中网络出现了问题，随着这个问题，我觉得我非常有必要去掌握 Traefik 这个组件。

## 问题描述

集群环境是1 Server 2 Agent 的模式，运行了一个简单地 Nginx 应用，整体结构如下：

- Deployment 中配置3个 Replica Pod,容器是 Nginx，挂载了本地存储内容是一个简单的前端内容。
- Deployment 前配置 Service,匹配上面 Pod 的标签，配置为 ClusterIP，暴露 Pod 给集群
- Service 前配置 Ingress ，将特定路径映射到 Service，为流量提供路由。
- IngressController 是这个结构最前面的部分，在 k3s 中默认是 Traefik

![1](/img/riscv/traefik01.png)

就当我访问 Ingress 中定义好的路径时，返回 `404 Pag Not Found`,显然这中间有地方出问题了。

- 使用 `ping <url>` 发现这个 url 的 IP 地址就是 Server 的地址且可以 ping 通，说明笔记本与节点的通信没问题。
- 检查 Deployment, Pod, Service, Ingress 的状态，都是正常运行，没有错误信息。
- 检查 Traefik 的日志，出现问题，于是检查这个问题。

## 问题所在

```bash
# 获取Traefik Pod 信息
sudo kubectl get pods -o wide -n kube-system -l app.kubernetes.io/name=traefik
# 查看日志信息
sudo kubectl logs <traefik_pod_name> -n kube-system
```

查看日志才发现，错误就在这里。

```bash
W0602 12:19:21.743160       1 reflector.go:561] k8s.io/client-go@v0.31.1/tools/cache/reflector.go:243: failed to list *v1.ConfigMap: configmaps is forbidden: User "system:serviceaccount:kube-system:traefik" cannot list resource "configmaps" in API group "" at the cluster scope
E0602 12:19:21.743322       1 reflector.go:158] "Unhandled Error" err="k8s.io/client-go@v0.31.1/tools/cache/reflector.go:243: Failed to watch *v1.ConfigMap: failed to list *v1.ConfigMap: configmaps is forbidden: User \"system:serviceaccount:kube-system:traefik\" cannot list resource \"configmaps\" in API group \"\" at the cluster scope" logger="UnhandledError"
```

经过一番 google，发现一个与我遇到[相同问题](https://forums.suse.com/t/k3s-traefik-3-installation-missing-clusterroles/45128)的人,他对于此问题的描述是 traefik 的配置文件中关于某些组件的定义有缺失，需要补充 traefik 的配置文件。

解决如下：`sudo kubectl edit clusterrole traefik-kube-system`
在其中添加 `discovery.k8s.io,nodes,endpointslices,serverstransporttcps`等字段，最终结果如下:

```bash
rules:
- apiGroups:
  - extensions
  - networking.k8s.io
  - discovery.k8s.io
  resources:
  - ingressclasses
  - ingresses
  - endpointslices
  verbs:
  - get
  - list
  - watch

- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - secrets
  - configmaps
  - nodes
  verbs:
  - get
  - list
  - watch

- apiGroups:
  - extensions
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - traefik.io
  resources:
  - ingressroutes
  - ingressroutetcps
  - ingressrouteudps
  - middlewares
  - middlewaretcps
  - tlsoptions
  - tlsstores
  - traefikservices
  - serverstransporttcps
  - serverstransports
  verbs:
  - get
  - list
  - watch
```

删除这个 Pod 之后它会自动重新生成，之后的结果就正常了，可以成功访问定义的路径。

## 具体原因

从 `forbidden` 可以看出，这其实是一个 [RABC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (Role-Based Access Control) 权限问题.

- Role 会定义有关于资源的操作权限（可以对哪些资源执行哪些操作）
- RoleBinding 会将这个定义好的权限给某个或某组用户(要注意特定命名空间)
- ServiceAccount: 集群内部的进程(例如 Pod 中运行的应用)

报错日志告诉我们，`User \"system:serviceaccount:kube-system:traefik\" cannot list resource \"configmaps\" in API group \"\" at the cluster scope"`即某个用户的对于某个资源的某个操作失败了。

即我们的 Traefik 在读取某些资源的时候失败了，权限不够！这会造成什么后果呢，请继续往下看。

## Traefik 在 K8s/K3s 中的应用

Traefik 到底是做什么的呢？(类似于 Nginx,但是又有其特点)

- 监视 (Watch) K8s/K3s 集群中的 Ingress、Service、EndpointSlice 等资源的变化
- 根据这些资源的信息，**动态配置其内部的路由规则**
- 将外部传入的流量路由到正确的后端 Service 和 Pod

在官网的[这篇文档](https://doc.traefik.io/traefik/getting-started/quick-start-with-kubernetes/)中可以看到 Traefik 在 K8s 中的应用。
值得一提的是，K3s 内置了 Traefik 作为 Ingress Controller,而 [K8s](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) 则是没有内置，默认维护 AWS,nginx 等 Controller，同时也可以采用第三方例如 Traefik.

在 K8s 中使用 Traefik,我们需要进行以下操作:

- 创建 role:ClusterRole
- 创建 ServiceAccount,将 role 绑定到 account 上；
- 部署 traefik deployment，设置 traefik 容器端口web 80,dashboard 8080
- 部署 traefik service,并在其中对应上面两个端口

而 K3s 中，观其源码可以发现，其在 `/manifest` 目录下定义了使用 Helm Controller 来管理例如 traefik 这样的预装组件，通过 Helm Chart 自动部署。

接下来我们来验证一下 K3s 集群中的 Traefik 属性:

**role & ServiceAccount**

```bash
# ServiceAccount,名称为 traefik
sudo kubectl get serviceaccount -n kube-system traefik
NAME      SECRETS   AGE
traefik   0         18d
# ClusterRole,名称为traefik-kube-system    
sudo kubectl get clusterrole | grep traefik
traefik-kube-system                                                    2025-05-15T06:59:41Z
# 查看其 Role 信息：
sudo kubectl get clusterrole traefik-kube-system -o yaml
```

这里查看到的信息就是我们上面修改好的“配置文件”,定义了 Traefik 可以访问哪些 apiGroups,groups 中的 resources,可以对这些 resources 进行哪些 verbs

**binding**

```bash
# 查看 clusterrolebinding 绑定信息
sudo kubectl get clusterrolebinding | grep traefik
helm-kube-system-traefik                                        ClusterRole/cluster-admin                                                   19d
helm-kube-system-traefik-crd                                    ClusterRole/cluster-admin                                                   19d
traefik-kube-system                                             ClusterRole/traefik-kube-system                                             18d
# 查看具体的绑定规则
sudo kubectl get clusterrolebinding traefik-kube-system -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    meta.helm.sh/release-name: traefik
    meta.helm.sh/release-namespace: kube-system
  creationTimestamp: "2025-05-15T06:59:41Z"
  labels:
    app.kubernetes.io/instance: traefik-kube-system
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: traefik
    helm.sh/chart: traefik-27.0.201_up27.0.2
  name: traefik-kube-system
  resourceVersion: "4041"
  uid: 905de4bb-8aab-42c6-a84c-169252444073
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-kube-system
subjects:
- kind: ServiceAccount
  name: traefik
  namespace: kube-system
```

在最后的部分可以发现就是把 traefik-kube-system 绑定到了 traefik 上，这与[官网](https://doc.traefik.io/traefik/getting-started/quick-start-with-kubernetes/)给出的示例几乎一致。

**Deployment & Service**

```bash
# Deployment
sudo kubectl get deployment -n kube-system traefik
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
traefik   1/1     1            1           18d
# Deployment 内部 Port 信息
ports:
        - containerPort: 9100
          name: metrics
          protocol: TCP
        - containerPort: 9000
          name: traefik
          protocol: TCP
        - containerPort: 80
          name: web
          protocol: TCP
        - containerPort: 8443
          name: websecure
          protocol: TCP
# Service
sudo kubectl get service -n kube-system traefik
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
traefik   LoadBalancer   10.43.144.33   192.168.31.70   80:32194/TCP,443:31405/TCP   18d
# Service 内部 Port 信息，与上面 Deployment 要对应
ports:
  - name: web
    nodePort: 32194
    port: 80
    protocol: TCP
    targetPort: web
  - name: websecure
    nodePort: 31405
    port: 443
    protocol: TCP
    targetPort: websecure
```

其实从这里与之前的集群图片对照，可以看出 Traefik Deployment 作为 Ingress Controller,其 Service 在前面作为 LoadBalancer,让外部的流量可以到达 Traefik Deployment.

![1](/img/riscv/traefik01.png)

## 与 Nginx 对比

从前面的介绍可以看出，Traefik 最大的卖点就是*实时监测集群内的路由情况并自动进行服务发现并为其动态配置路由。*

而传统的 Nginx 想要完成这样的自动化是不可能的，需要手动更改 `nginx.conf`文件，向其中添加路由规则，所以其并不适合动态服务。其通常用作静态文件服务和反向代理。

但是 Nginx 也像 Traefik 一样，提供了动态监听的 Ingress Controller: **Nginx Ingress Controller**,虽然 K3s 默认嵌入 Traefik,但如果你想，也可以替换成 Nginx.

## 总结

Traefik 作为集群中的 Ingress Controller,需要调用集群中的资源来为我们后方 Deployment Pod 中的应用**自动**构建路由规则，将外部的请求转发过来。

所以我们需要保证其有访问某些资源的权限，例如 `nodes,endpointslices` 等资源。

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**