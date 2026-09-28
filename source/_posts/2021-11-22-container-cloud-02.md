---
layout: post
title: "容器与云计算（二）：容器隔离的四个 Linux 基础"
date: 2021-11-22 20:00:00 +0800
comments: true
categories: cloud-native linux
---

容器看起来像一台独立的机器，实际上只是由 Linux 内核提供了隔离视图和资源控制的一组进程。理解 Namespace、cgroup、文件系统和网络，才能知道容器的边界在哪里。

<!--more-->

## 一、Namespace：让进程看到不同的世界

Namespace 为进程提供隔离的系统视图。常见类型包括：

* PID Namespace：容器内的第一个进程可以看到自己是 PID 1。
* Mount Namespace：容器可以拥有自己的挂载点和根文件系统。
* Network Namespace：容器拥有独立的网卡、路由表和端口空间。
* UTS Namespace：可以设置独立的主机名与 NIS 域名（domain name）。
* IPC、User Namespace：分别隔离进程间通信对象和用户/组 ID。

Namespace 解决的是“能看到什么”。它不是完整的安全边界：宿主机内核仍然被共享，特权配置、内核漏洞和错误的设备访问都可能扩大影响面。

## 二、cgroup：限制进程能用多少

cgroup 解决的是资源控制和统计。它可以限制或记录 CPU、内存、块设备 IO、进程数量等资源。容器内的进程即使看到一台机器，也只能在自己的资源配额内运行。现在主流 Linux 发行版通常优先使用 cgroup v2；排查资源问题时，应确认宿主机和运行时实际使用的层级，而不是只看容器命令行参数。

资源限制必须配合应用行为理解。例如内存限制触发后，内核可能杀死容器中的进程；CPU 限制会让进程获得更少的调度时间。线上设置限制时，应同时设置合理的请求值、上限值和告警指标。

## 三、镜像与 UnionFS：把文件系统分层

容器镜像通常由多层只读文件组成，运行容器时再叠加一层可写层。多个镜像可以共享相同的基础层，因此分发和存储成本比复制完整虚拟机低。Docker 常见的 `overlay2` 和其他运行时 snapshotter 都是在实现这一类分层文件系统模型。

镜像层是构建缓存和分发的基础，但可写层不适合保存重要业务数据。需要持久化的数据应该放到卷或外部存储，并明确备份、恢复和迁移策略。

## 四、容器网络：从虚拟网卡到服务端口

容器网络通常通过 Network Namespace、虚拟以太网设备、网桥和 NAT 组合实现。容器看到的是自己的网卡，宿主机通过网桥连接多个容器，再通过端口映射把服务暴露出去。

端口映射只解决“如何到达”，不解决服务发现、负载均衡和访问控制。在多实例环境中，这些职责通常交给编排平台或独立的网络组件。

## 五、容器的真实边界

容器提供的是进程级隔离，不是魔法安全盒。生产环境至少要做到：使用非 root 用户、减少 Linux capabilities、只读根文件系统、限制资源和系统调用、及时更新宿主机内核与镜像，并扫描第三方依赖。

理解这些机制后，Docker 的命令就不再是孤立的记忆题：`run` 创建隔离的进程环境，`--memory` 和 `--cpus` 设置资源约束，`-p` 配置端口映射，`-v` 连接持久化存储。

下一篇会从镜像、容器、网络、卷和 Compose 说明如何把这些内核能力组织成可复现的开发与部署流程。

参考资料：

* [Docker security](https://docs.docker.com/engine/security/)
* [Linux namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)
* [Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
