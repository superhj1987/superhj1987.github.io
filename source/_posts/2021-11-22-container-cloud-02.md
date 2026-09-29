---
layout: post
title: "容器与云计算（二）：容器隔离的四个 Linux 基础"
date: 2021-11-22 20:00:00 +0800
comments: true
categories: cloud-native linux
---

> 内容更新于 2026-09-29。保留原发布日期，正文与示例已按更新时的官方文档修订。

在容器里面执行 `ps`，只能看到几个进程；查看根目录，又像进入了一套独立系统。可是在宿主机上，这些应用仍然是普通的 Linux 进程。容器究竟在哪里建立了边界？

答案不是某一个神奇的系统调用，而是多种内核机制的组合：Namespace 改变进程看到的系统视图，cgroup 管理资源使用，文件系统挂载组织应用环境，网络机制连接进程与外部世界。理解这些基础以后，资源限制、端口映射和容器逃逸等问题就能放到同一套模型中分析。

<!--more-->

## 一、Namespace：同一个内核，不同的视图

一个进程能看到哪些进程、挂载点、网络设备和主机名，取决于它所在的 Namespace。容器运行时创建或加入相应的命名空间，再在其中启动应用，于是应用获得了类似独立机器的视图。

| 类型 | 隔离对象 | 对容器的意义 |
|---|---|---|
| PID | 进程 ID 空间 | 容器内外的同一进程可能有不同 PID |
| Mount | 挂载点视图 | 每个容器可以看到不同的根目录和挂载 |
| Network | 网卡、路由、端口等 | 不同容器可以分别监听相同端口 |
| UTS | 主机名、NIS 域名 | 容器可设置自己的主机标识 |
| IPC | 部分进程间通信资源 | 隔离 System V IPC、POSIX 消息队列等 |
| User | 用户和组 ID 映射 | 容器内 root 可以映射为宿主的普通用户 |

这不是完整的 Namespace 清单；例如 cgroup Namespace 还可以隔离进程看到的 cgroup 路径视图。这里先关注直接影响应用行为的几类。

### 1.1 PID 1 不只是一个编号

容器内的主进程经常是 PID 1，它需要正确处理退出信号和回收被收养的孤儿进程。如果应用通过 shell 脚本启动，而 shell 没有把信号传给子进程，停止容器时就可能一直等到超时，最终被强制终止。

因此 Dockerfile 通常使用 exec 形式启动主进程，例如 `ENTRYPOINT ["java", "-jar", "/app/app.jar"]`。需要 shell 做初始化时，最后用 `exec` 替换 shell 进程。对于会产生子进程的应用，还可以按需使用轻量 init，但这不能替代应用本身的优雅退出逻辑。

### 1.2 文件系统隔离不等于 chroot

`chroot` 改变路径查找时使用的根目录，本身不负责隔离进程、网络或资源，也不适合作为完整的安全边界。容器运行时会结合 Mount Namespace、根文件系统切换和权限控制，构造应用所见的文件系统。

两个容器都看到 `/etc`，不表示它们访问同一目录；而把宿主目录 bind mount 进去以后，容器就可能直接修改宿主上的文件。看到的路径只是表象，排查时需要继续确认挂载来源、读写权限和挂载传播行为。

### 1.3 一个可以观察的实验

以下命令在已有 Docker Engine 的实验环境执行。观察宿主 `/proc` 的步骤适用于原生 Linux；Docker Desktop 的宿主内核位于其 Linux 虚拟机内。

{% raw %}
```plain
docker run -d --name ns-demo alpine:3.21 sleep 600
docker exec ns-demo sh -c 'echo "hostname=$(hostname)"; ps'
docker inspect --format '{{.State.Pid}}' ns-demo
```
{% endraw %}

最后一条返回容器主进程在 Docker 宿主上的 PID。将结果代入下面路径，可以观察它所属的命名空间：

```plain
# 把 12345 换成上一步返回的宿主 PID
sudo ls -l /proc/12345/ns/
ls -l /proc/self/ns/
```

符号链接中的编号可以帮助判断两个进程是否共享某类命名空间。它也解释了为什么容器没有绝对统一的隔离方式：使用 host 网络等配置时，进程会与宿主共享特定 Namespace。实验完成后执行 `docker rm -f ns-demo`。

## 二、cgroup：资源上限与争用行为

Namespace 主要回答“看见什么”，cgroup 则回答“如何分配、限制和统计资源”。CPU、内存、进程数量和 IO 都可以纳入控制，但前提是内核、控制器和运行时具备对应支持。

cgroup v2 使用统一层级组织进程与资源控制器。不要只根据发行版名称猜测实际配置，可以检查 `/sys/fs/cgroup/cgroup.controllers` 是否存在，以及 `docker info` 中的 cgroup 信息。cgroup Namespace 隔离的是视图，而 cgroup 控制器执行的是资源管理，两者不是同一个功能。

### 2.1 CPU 限制不是给进程绑一颗核心

例如把容器限制为 0.5 CPU，表达的是一段时间内可获得的 CPU 时间预算，通常不表示只允许它运行在某一颗核心上。在多核环境中，多个线程可以短时间并行消耗配额，然后在剩余周期内被节流。

在 cgroup v2 中，`cpu.max` 用配额和周期表达 CPU 带宽上限；`cpu.weight` 表达竞争时的相对权重；允许在哪些 CPU 上运行，则属于 cpuset 等机制的职责。

这对 Java 服务很重要：线程多不等于可用 CPU 多。CPU limit 太紧时，GC 和请求处理一起消耗预算，即使宿主机整体 CPU 不高，应用也可能出现延迟尖峰。排查需要同时看进程负载、CPU 节流统计和响应时间。

### 2.2 内存限制与 OOM

内存不像 CPU 时间那样可以简单等待下一个调度周期。达到限制时，内核会尝试回收；无法满足分配时，可能触发 cgroup 内的 OOM 处理并杀死进程。`memory.current`、`memory.max` 和 `memory.events` 是 cgroup v2 下常用的观察点。

对 JVM 而言，堆只是进程内存的一部分。线程栈、元空间、直接缓冲区、JIT 代码缓存和本地库也会消耗内存。把 `-Xmx` 设置成容器内存上限，容易导致“堆还没满，进程却被杀”的现象。

退出码 137 通常表示收到 SIGKILL，但不能单凭它认定一定是 OOM。应结合容器状态、内核日志和 cgroup 事件确认原因；管理员手工强制结束进程也会产生相似结果。

### 2.3 观察资源约束

{% raw %}
```plain
docker run -d --name limits-demo \
  --memory=256m --cpus=0.5 --pids-limit=128 \
  alpine:3.21 sleep 600

docker stats --no-stream limits-demo
docker inspect --format '{{json .HostConfig}}' limits-demo
docker exec limits-demo cat /proc/self/cgroup
```
{% endraw %}

在标准 cgroup v2 挂载环境中，还可以查看容器可见路径下的 `cpu.max`、`memory.max` 等文件；具体路径取决于 cgroup Namespace 和挂载设置。这个实验验证配置与观察方式，不需要故意把主机压到资源耗尽。结束后执行 `docker rm -f limits-demo`。

## 三、文件系统：镜像层和运行状态

### 3.1 层是怎么组合起来的

镜像通常通过多层文件系统差异描述内容。启动容器时，运行时在镜像内容之上提供可写层，让应用看到一个统一的目录树。多个容器可以复用只读内容，而各自的修改互不影响。

OverlayFS 是常见的实现基础：lower 层提供已有内容，upper 层记录修改，merged 挂载提供组合视图。修改下层文件时可能触发 copy-up，将文件复制到可写层后再改写；删除操作则通过相应标记遮蔽下层内容。

所以，从后续镜像层删除密钥文件，不代表它已经从早期层中消失。构建时就不能把密钥放入镜像层，清理动作无法可靠补救错误的构建流程。

### 3.2 不要把 overlay2 当作所有环境的固定答案

Docker 的存储后端取决于版本和配置，可能使用经典存储驱动，也可能使用 containerd image store 与 snapshotter。需要先看 `docker info` 和实际启用的后端，再分析磁盘布局，不能照着旧教程硬套某个内部目录。

镜像采用分层分发，与运行时用哪一种方式准备可挂载文件系统，是相关但不同的两个问题。理解模型比记住某个目录名更有用。

### 3.3 卷、bind mount 与 tmpfs

| 方式 | 适合用途 | 需要关注 |
|---|---|---|
| 容器可写层 | 可丢弃的运行时文件 | 随容器删除，可能有写时复制开销 |
| Volume | 与容器生命周期分离的数据 | 仍需备份，并确认存储所在故障域 |
| Bind mount | 开发源码、指定配置或宿主目录 | 路径耦合及宿主文件权限 |
| tmpfs | 无须持久化的临时数据 | 占用内存，实例停止后数据丢失 |

Volume 独立于容器，不等于独立于宿主机。一台机器磁盘损坏，本地卷仍然可能全部丢失。持久化、复制和备份是三件不同的事，不能用“已经挂卷”代替恢复设计。

## 四、网络：一条请求如何到达容器

在常见的 Linux bridge 网络中，容器拥有独立 Network Namespace，通过 veth 设备连接宿主网桥。容器侧负责收发数据，宿主侧再把流量送往同一网络中的其他容器或外部网络。

发布端口会建立从宿主端口到容器端口的访问路径。具体转发可能涉及防火墙规则、NAT 或代理，并会随平台和网络后端变化；不能假设所有环境都经过完全相同的 iptables 链。

### 4.1 localhost 到底是谁

普通 bridge 网络里的两个容器分别拥有自己的回环接口。应用容器连接 `localhost:5432`，访问的是自身网络空间，而不是另一个 PostgreSQL 容器。通常应通过同一用户自定义网络上的服务名连接数据库。

宿主端口与容器监听地址也有区别。应用只监听容器内的 `127.0.0.1` 时，从容器网卡方向到达的请求通常无法访问它，即使端口已经发布。需要对外提供服务的应用一般监听容器接口或 `0.0.0.0`，再由外层网络策略限制访问范围。

```plain
docker network create demo-net
docker run -d --name web-demo --network demo-net \
  -p 127.0.0.1:8080:80 nginx:stable-alpine
curl http://127.0.0.1:8080/
docker run --rm --network demo-net alpine:3.21 \
  wget -qO- http://web-demo/
```

这里将宿主发布地址限制在回环接口，仅用于本地实验。同一 Docker 网络内的访问则使用容器名和容器端口。完成后先删除 `web-demo` 容器，再删除 `demo-net` 网络；生产镜像版本固定方法见下一篇。

### 4.2 网络故障按路径排查

先确认应用是否在监听，再确认监听地址和端口；随后测试同网络容器之间的可达性，最后检查宿主端口发布、路由和防火墙。域名无法解析与端口拒绝连接属于不同层次，不应靠反复重启容器来碰运气。

容器网络也不必一定经过 bridge 和 NAT。host、macvlan、overlay，以及 Kubernetes 的不同 CNI 实现，都可能采用不同的数据路径。故障定位的第一步是识别当前使用的模型。

## 五、把隔离机制变成安全边界

Namespace 与 cgroup 提供了重要基础，却不保证恶意代码绝对无法影响宿主机。常规 Linux 容器共享内核，内核漏洞、危险设备访问、特权配置和宿主目录挂载，都可能扩大攻击面。

非 root 用户降低进程权限；User Namespace 进一步改变容器用户与宿主用户的映射；rootless 模式还让运行时尽量以普通用户身份工作。这些措施职责不同，不能因为容器内 UID 不是 0 就认为完成了全部隔离。

实际部署应按应用需要减少 capabilities、限制提权，使用 seccomp 和 AppArmor/SELinux 等机制约束操作，并保持内核和镜像更新。只读根文件系统有助于减少任意写入，但需要为必要的临时目录单独提供可写空间。

尤其不要轻易把 Docker socket 挂到业务容器。能够控制高权限 Docker daemon，往往意味着能够创建带宿主挂载或更高权限的新容器，其风险远大于普通业务 API 权限。

对不可信多租户代码，还需要评估虚拟机、沙箱运行时和独立节点等额外边界。隔离强度、兼容性和性能成本应根据威胁模型决定，而不是只根据启动速度决定。

[上一篇：从物理机到云原生](/blog/2021/05/15/container-cloud-01/) · [下一篇：用 Docker 建立可重复交付](/blog/2022/06/18/container-cloud-03/)

## 参考资料

- [Linux namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [Docker 资源限制](https://docs.docker.com/engine/containers/resource_constraints/)
- [Docker 存储概览](https://docs.docker.com/engine/storage/)
- [Docker bridge 网络](https://docs.docker.com/engine/network/drivers/bridge/)
- [Docker 安全模型](https://docs.docker.com/engine/security/)
