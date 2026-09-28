---
layout: post
title: "容器与云计算（四）：Kubernetes 的核心模型"
date: 2022-09-26 20:00:00 +0800
comments: true
categories: cloud-native kubernetes
---

Docker 解决了单机上的打包和运行；当容器数量增加到多台机器时，还需要调度、服务发现、滚动升级和故障恢复。Kubernetes 的核心思想是：用户声明期望状态，控制器持续把实际状态调整到期望状态。

<!--more-->

## 一、Pod 是调度的最小单位

Pod 包含一个或多个紧密协作的容器。Pod 内的容器共享网络命名空间，可以通过 `localhost` 通信，也可以共享卷。通常一个 Pod 只运行一个主应用，Sidecar 只有在确实需要共享生命周期或本地资源时才加入。

Pod 是短暂的。它被删除或重新调度后，IP 和本地文件都可能变化，因此应用不能把 Pod 身份当成稳定地址或持久化存储。

## 二、Deployment 负责无状态应用

Deployment 描述一组相同的 Pod 副本，并通过 ReplicaSet 完成创建、替换和滚动更新。更新镜像时，控制器逐步创建新版本并减少旧版本，期间可以通过就绪探针决定哪些实例能够接收流量。

副本数解决可用性和吞吐问题，但不等于高可用。应用还需要跨节点分布、合理的资源请求、优雅关闭和数据层的容错设计。

## 三、Service 提供稳定访问入口

Service 为一组符合标签选择器的 Pod 提供稳定的虚拟地址和端口。Pod 发生替换时，Service 仍然保持不变，后端端点由控制器自动更新。

集群内部服务可以使用 ClusterIP；需要从集群外部访问时，可以使用 Gateway 或云厂商提供的负载均衡能力。Ingress 仍然常见，但新系统应根据 Kubernetes 版本和平台能力评估 Gateway API 等方案。

## 四、资源、探针和配置

容器应声明 CPU、内存等资源请求和上限。调度器根据请求选择节点，运行时根据上限限制资源。请求过低会导致节点过度装载，上限过高则会降低集群利用率。

存活探针用于判断进程是否需要重启，就绪探针用于判断实例是否可以接收流量，启动探针用于保护启动较慢的应用。配置和敏感信息应分别使用 ConfigMap 和 Secret，并通过权限控制限制读取范围。

## 五、控制器思维

Kubernetes 中的对象是声明，控制器是实现闭环的程序。Deployment、Job、StatefulSet 和自定义 Operator 都遵循相同的思路：读取当前状态，比较期望状态，然后执行小步调整。

这种模型带来自动恢复和可扩展性，也带来新的复杂度：状态是异步收敛的，删除和更新可能不是立即完成的，故障排查必须结合事件、日志、指标和对象状态，而不能只看一次命令输出。

## 六、生产落地的最低要求

生产集群至少需要明确镜像来源、RBAC 权限、网络策略、资源配额、备份恢复、升级策略和审计方式。对有状态服务，优先评估托管服务；确实运行在集群内时，要先验证故障转移、数据恢复和跨可用区能力。

Kubernetes 是基础设施抽象层，不会替应用自动获得高可用。只有应用、数据、网络和发布流程都能处理失败，系统才真正具备云原生的弹性。

参考资料：

* [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
* [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
* [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
* [Gateway API](https://gateway-api.sigs.k8s.io/)
