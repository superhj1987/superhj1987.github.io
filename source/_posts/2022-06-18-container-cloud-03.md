---
layout: post
title: "容器与云计算（三）：用 Docker 建立可重复交付"
date: 2022-06-18 20:00:00 +0800
comments: true
categories: cloud-native devops
---

> 内容更新于 2026-09-29。保留原发布日期，正文与示例已按更新时的官方文档修订。

一个应用在开发电脑上能运行，并不代表能够稳定交付。测试环境缺少某个系统库，生产环境使用另一版 JRE，发布时临时登录服务器修改配置，这些差异会让一次代码变更变成难以复现的现场操作。

Docker 的价值在于把运行环境和应用一起组织成制品，再让部署流程围绕这个制品展开。真正要建立的是一条可追溯的链路：源码版本对应哪个镜像，镜像经过哪些验证，部署到了哪里，故障时能否回到已知状态。

<!--more-->

## 一、区分源码、镜像和运行实例

镜像包含应用文件、用户空间依赖和默认运行配置；容器是在特定资源、网络、挂载和权限配置下运行的实例。同一个镜像可以启动多个容器，它们共享制品内容，却各自拥有运行状态。

这里最容易混淆的是标签和摘要。`orders:release-42` 是人类容易理解的名字，但仓库中的标签可以被重新指向；`orders@sha256:...` 用摘要标识具体内容。发布记录只写标签，不能保证以后取回的还是同一个制品。

因此，一次发布最好同时记录 Git commit、构建流水线编号、镜像摘要和部署配置版本。镜像的不可变性由内容标识提供，标签是否不可变还需要仓库策略保障。

### 1.1 同一个镜像也不等于相同环境

测试与生产使用相同镜像，只能保证制品一致。数据库版本、外部服务、网络限制和 CPU 架构仍可能不同。多架构镜像还可能通过一个镜像索引关联多个平台的镜像，实际运行内容需要结合目标平台理解。

容器化的目标是缩小变量范围，让差异明确且可管理，而不是承诺“一次打包，无条件到处运行”。

## 二、从一个 Java 项目构建镜像

下面以使用 Maven 的单模块 Spring Boot 项目为例。约定运行 Java 21，`pom.xml` 将可执行 jar 的最终名称设为 `app.jar`，构建结果位于 `target/app.jar`。已有项目需要根据自己的模块结构和插件配置调整路径。

这里的版本标签用于说明构建流程，并非“最新版本”推荐。正式使用时，应选择组织维护的补丁版本或经过验证的摘要，并安排定期更新。

```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace

COPY pom.xml ./
RUN --mount=type=cache,target=/root/.m2 mvn -B dependency:go-offline

COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -B verify

FROM eclipse-temurin:21-jre-jammy AS runtime
RUN groupadd --gid 10001 app \
    && useradd --uid 10001 --gid app --no-create-home app
WORKDIR /app
COPY --from=build --chown=app:app /workspace/target/app.jar ./app.jar
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

第一阶段准备 Maven 和 JDK，执行项目验证并生成制品；第二阶段只保留 JRE 与应用 jar。运行镜像不必包含源码、编译器和 Maven 缓存，职责更清楚，也减少了需要维护的文件。

`EXPOSE` 是端口说明，不会自动发布端口。执行 `docker run -p` 或交给编排平台配置 Service 后，才建立相应的访问入口。

### 2.1 构建缓存为什么要分层组织

先复制 `pom.xml`，再复制源代码，是为了让依赖准备层不因每次业务代码变化都失效。BuildKit 的缓存挂载则让不同构建步骤复用 Maven 下载缓存；它不是应用运行数据，也不应包含进入镜像的凭证。

`dependency:go-offline` 能预取许多依赖，但复杂插件、动态版本或特殊模块结构仍可能在后续构建时访问网络。可重复构建还需要锁定依赖、插件和基础镜像，并控制外部制品来源，不能仅靠 Docker 层缓存保证。

构建上下文也应尽量小。对这个示例，可以使用：

```text
.git
.idea
.vscode
target
*.log
.env
.env.*
```

这是 `.dockerignore`，用于阻止无关文件进入构建上下文。如果构建本来就需要复制预先生成的 `target`，则应调整规则。忽略文件的作用是减少输入和意外暴露，不是代替仓库中的密钥治理。

### 2.2 构建和本地运行

```bash
docker buildx build --load -t orders:local .
docker run --rm --name orders-local \
  -p 127.0.0.1:8080:8080 \
  --memory=768m --cpus=1 \
  --read-only --tmpfs /tmp:rw,nosuid,size=64m \
  orders:local
```

`--load` 将单平台构建结果加载到本地 Docker，方便实验；向仓库交付通常使用 `--push`。多平台构建还需要验证本地库和基础镜像是否支持目标架构，不能只因为构建命令成功就认为应用兼容。

示例假设应用只向 `/tmp` 写临时文件，业务配置已经能够满足启动条件。需要数据库、上传目录或额外配置时，应显式提供对应资源，而不是取消所有限制来让启动暂时成功。

## 三、配置和密钥什么时候进入系统

镜像应该表达应用版本，部署配置表达它在某个环境中的运行方式。数据库地址、日志级别和功能开关通常不需要重新编译代码；它们可以由环境变量或挂载文件注入。

密钥则需要更严格的处理。不要通过 Dockerfile 的 `ARG` 或 `ENV` 写入仓库密码，也不要先 `COPY` 凭证再删除。构建历史、层内容或元数据都可能留下可恢复的信息。

BuildKit 提供 secret mount，让某个构建步骤临时读取凭证：

```dockerfile
RUN --mount=type=secret,id=maven_settings,target=/root/.m2/settings.xml \
    --mount=type=cache,target=/root/.m2 \
    mvn -B verify
```

对应构建命令通过 `--secret id=maven_settings,src=/安全路径/settings.xml` 提供文件。私有依赖下载的其他步骤也要配置相同凭证挂载。应用和构建工具仍不能把密钥输出到日志或复制进产物，secret mount 不会阻止错误脚本主动泄露内容。

运行时密钥可以从受控文件或密钥管理服务提供。环境变量虽然方便，但有可能出现在诊断输出、进程信息或配置检查中，应结合平台权限和审计方式决定是否适用。

## 四、进程退出和健康检查是交付协议的一部分

### 4.1 应用应该怎样退出

停止容器时，运行时通常先发送终止信号，等待约定时间，超时后再强制结束。应用需要停止接收新工作，等待在途请求或任务完成，关闭连接，并在时间预算内退出。

这也是 exec 形式 `ENTRYPOINT` 的意义：让 Java 成为直接接收信号的主进程，避免多余 shell 破坏信号传递。对 Spring Boot 还应根据使用的版本配置优雅关闭，并确保平台终止宽限期能够覆盖应用实际关闭耗时。

后台消费者尤其需要注意：接收终止信号以后，应停止拉取新消息，再完成或放弃当前任务。任务即使可能被重复投递，也应依靠业务幂等保持正确，不能寄希望于容器永远不会被强制终止。

### 4.2 健康不只有“进程还在”

进程存活、能够接收请求、依赖系统可用，是不同状态。一个应用可能在初始化缓存，也可能因为线程池耗尽而无法处理业务。只检查进程 ID 无法判断这些情况。

Docker 的 `HEALTHCHECK` 可以记录健康状态，但 Docker Engine 并不会仅因为状态变成 unhealthy 就自动按重启策略重启容器；重启策略主要处理进程退出。Kubernetes 则需要单独配置自己的探针，不会直接把 Dockerfile 的健康检查当作 Pod 探针。

检查逻辑应有超时、合理频率和清晰用途。把数据库短暂波动直接转成全部应用实例的重启，很容易让故障扩大。

## 五、用 Compose 描述一组协作服务

当应用还依赖数据库、缓存或消息队列时，单个 `docker run` 无法方便地表达整个开发环境。Compose 把服务、网络和卷放在一个声明文件里，适合团队共享本地或单机部署配置。

下面仅演示 PostgreSQL 与 Redis 两个开发依赖。PostgreSQL 使用明确的大版本，避免跨大版本时直接复用数据目录；示例不是生产数据库部署方案。

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: orders
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD}
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders -d orders"]
      interval: 5s
      timeout: 3s
      retries: 10
  cache:
    image: redis:7-alpine
    ports:
      - "127.0.0.1:6379:6379"
volumes:
  pgdata:
```

Redis 在这里被视为可丢弃缓存，没有配置数据持久化。数据库密码需要在本地环境中设置，不能把真实生产密码写入版本库；如果通过 `.env` 提供，也需要排除该文件。

执行 `docker compose up -d` 后，可以用 `docker compose ps` 查看状态、用 `docker compose logs db` 查看数据库日志。运行在宿主上的 Java 服务访问回环地址；如果把应用也加入这个 Compose 项目，则应使用 `db:5432` 和 `cache:6379`，因为 Compose 会建立项目网络和服务发现。

### 5.1 启动顺序不等于业务永远可用

普通 `depends_on` 主要描述启动依赖。需要等待依赖健康时，可以在长语法中使用 `condition: service_healthy`，并为依赖提供 healthcheck。但这只是启动阶段的协调，数据库运行过程中仍会重启或短暂不可达。

应用必须具备连接超时、重连和合理的失败处理。健康检查也不能自动完成数据库 schema 迁移，迁移仍应作为可追踪的部署步骤。

### 5.2 Compose 也可以用于单机生产

把 Compose 说成“只能开发使用”并不准确。对于规模有限、单机故障风险可以接受的服务，它可以作为生产部署工具，并通过单独配置处理镜像、资源和重启策略。

它的边界在于没有提供 Kubernetes 那样的跨节点调度与控制器体系。主机损坏后如何恢复、流量如何切换、备份怎样取回，仍需要其他机制。是否升级为集群平台，应由可用性目标和团队能力决定。

## 六、数据如何穿过发布和回滚

容器可以替换，订单数据不可以。数据库文件需要持久存储，用户附件应进入独立的文件或对象存储，日志尽量输出到标准输出和标准错误，再由外部系统收集。

本地卷能跨越容器生命周期，却不能自动跨越宿主机故障。备份必须有明确的保留策略和恢复演练；复制可以提高可用性，却也可能同步误删操作，因此不能替代备份。

数据库升级也是回滚的边界。新版本应用把某个字段删除后，回退旧镜像可能已经无法工作。更稳妥的方式是分阶段演进：先增加兼容的新结构，再发布支持新旧结构的应用，完成数据迁移后，最后移除旧结构。这需要在发布设计阶段完成，而不是出故障后临时补救。

## 七、一条可追溯的流水线

可重复交付至少包含以下阶段：

1. 从确定的源码提交构建，运行项目测试和检查。
2. 生成镜像，执行制品扫描，并按需要产生 SBOM 和构建来源证明。
3. 推送镜像，记录摘要以及依赖、基础镜像和构建配置。
4. 将同一制品推进测试和生产环境，环境差异通过配置提供。
5. 先发布小范围实例，观察错误率、延迟和业务指标，再扩大范围。
6. 保留旧制品与部署记录，确认应用和数据是否支持回退。

BuildKit 是构建后端，Buildx 提供构建入口和能力管理；漏洞扫描需要相应的扫描工具，不能把“用了 Buildx”当作“完成了安全扫描”。镜像签名和来源证明也需要消费端的校验策略，生成了文件并不等于建立了信任链。

基础镜像摘要固定后还要定期更新。固定解决的是意外漂移，更新解决的是漏洞和维护问题；一直锁在旧摘要上同样会积累风险。

## 八、从 Docker 进入集群之前

Kubernetes 通常通过 CRI 对接 containerd、CRI-O 等容器运行时。镜像标准让 Docker 构建的镜像可以被其他兼容运行时使用；Docker CLI、Docker Engine 与 Kubernetes 的运行时接口分别处在不同层次。

进入集群之前，先验证应用的行为：能否只依赖镜像和外部配置启动？临时目录是否明确？是否正确处理终止信号？实例替换会不会丢数据？日志能否从容器外查看？这些基础没有做好，换成更复杂的编排系统只会让问题更难定位。

[上一篇：容器隔离的四个 Linux 基础](/blog/2021/11/22/container-cloud-02/) · [下一篇：Kubernetes 的核心模型](/blog/2022/09/26/container-cloud-04/)

## 参考资料

- [Docker 构建最佳实践](https://docs.docker.com/build/building/best-practices/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [BuildKit](https://docs.docker.com/build/buildkit/)
- [构建密钥](https://docs.docker.com/build/building/secrets/)
- [Compose 启动顺序](https://docs.docker.com/compose/how-tos/startup-order/)
- [Compose 生产部署](https://docs.docker.com/compose/how-tos/production/)
- [Kubernetes 容器运行时](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
