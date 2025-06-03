+++
date = '2025-05-24T19:45:03+08:00'
draft = true
title = '040K3sEP13关于Pod的一切'
+++

## 引子

对于 Pod 的一切规则

暂时以这些字段进行测试

Pod:

- 容器
- 卷
- 调度
- 生命周期
- 主机名，主机名字空间
- 服务账号
- 安全上下文

容器：

- 镜像
- Entrypoint
- 端口
- 环境变量
- 卷
- 资源
- 生命周期
- 安全上下文

## Lifetime 生命周期

UID

指数退避延迟

## 资源分配

### 为容器分配资源

requests & limits

如果要直接为 Pod 指定分配的资源，需要启用 `PodLevelResources` feature gate;所以还是为每个容器分配资源，最终资源总和就是这个 pod 资源总和。

### 调整资源

传统上，更改 Pod 的资源需求需要删除现有的 Pod 并创建一个替换品，这通常由工作负载控制器管理。

```bash
kubectl edit deploy xxx      # 修改 YAML 文件
kubectl rollout restart xxx  # 触发滚动重启
```

原地 Pod 调整大小允许在运行中的 Pod 内更改容器（s）的 CPU/内存分配，从而可能避免应用程序中断
只能调整 CPU 和内存，无法降低内存限制
但是只有高版本支持调整操作，所以调整操作在日常的使用中并不常见？

### Qos 服务质量

- Guaranteed:limits == requests
- Burstable:不等于
- BestEffort:没有任何要求和限制

查看指定字段信息

## 卷

### empty

- volumes
- volumeMounts

### 持久卷

- pv 持久卷
- pvc 持久卷声明:找到符合的 pv 进行绑定并独占
- Pod -> pvc -> pv -> 实际存储后端(NFS,hostPath,CephFS)

storageClassName 动态供应

### Projected Volume

投影卷，聚合多种资源，统一挂载路径

- secret
- configMap
- downardAPI
- serviceAccountToken

## 权限

### Security Context

securityContext 控制容器中运行用户、组、文件系统权限

- runAsUser：容器进程的用户ID
- runAsGroup：容器进程的主要组ID
- fsGroup：卷的文件系统组ID
- supplementalGroups：附加组

### Service Accounts

Pod 中的应用程序与 K8s API 交互；每个 namespace 都有一个 default 用户

为 Pod 提供精细的 API 访问权限(特定权限)