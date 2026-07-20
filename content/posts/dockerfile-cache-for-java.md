---
title: Java 镜像构建变慢时：把 Dockerfile 缓存拆成可验证的层
date: 2026-07-20T09:00:42+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Docker
  - DevOps
  - 工程实践
categories:
  - devops
toc: true
math: false
lightgallery: false
comment: false
---

Java 服务的 Docker 镜像构建慢，常常不是 Maven 本身慢，而是 Dockerfile 把**变化频率不同的文件**放进了同一层：改一行业务代码，就让依赖下载、打包和镜像组装全部重来。

优化的目标不是追求一次构建的最快数字，而是让日常代码变更只失效必要的层，并且能在 CI 日志中验证缓存是否真的命中。

<!--more-->

## 先理解：缓存失效会向后传递

Dockerfile 按顺序执行。某条 `COPY` 或 `RUN` 的输入变化后，该层及其后续层通常都需要重新执行。因此，下面这种写法很直观，却会让任意源码变更都影响依赖解析：

```dockerfile
FROM maven:3-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY . .
RUN mvn -B -DskipTests package
```

`COPY . .` 的范围通常远大于编译所需内容。README、IDE 配置、测试报告，甚至本地构建产物，都可能改变构建上下文。第一步不是调整 JVM 参数，而是先减少上下文，并把依赖描述文件放到源码之前复制。

## 一份可用的 Spring Boot Dockerfile

以下示例以 Maven 单模块项目为例。镜像标签由流水线传入，基础镜像版本应由团队根据自身支持策略固定和更新，不能把示例标签直接当作生产标准。

```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3-eclipse-temurin-21 AS build
WORKDIR /workspace

# 依赖变化较少：先复制，便于复用下载结果
COPY pom.xml ./
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -q -DskipTests dependency:go-offline

# 业务代码变化频繁：放在后面
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -DskipTests package \
    && cp target/*.jar /tmp/app.jar

FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system --uid 10001 appuser
COPY --from=build /tmp/app.jar app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

这里有三个边界：

1. `pom.xml` 变更才会导致依赖预热层失效；只改 `src/` 时，依赖层有机会复用。
2. `--mount=type=cache` 保存的是构建期 Maven 仓库，不会把 `.m2` 打进最终运行镜像。它依赖支持该语法的构建器；在 CI 上先确认实际构建命令启用了相应能力。
3. 最终阶段只复制 JAR，不带 Maven、源码和编译缓存。运行镜像更容易审计，也减少了无关文件进入生产环境的机会。

## `.dockerignore` 是缓存设计的一部分

即使 Dockerfile 分层正确，过大的上下文仍会拖慢上传并扩大误失效范围。项目根目录至少应有：

```gitignore
.git
.idea
.vscode
*.iml
target
README.md
Dockerfile*
docker-compose*.yml
```

不要机械照抄这份清单。例如构建脚本确实会读取 `README.md` 或 Compose 文件时，就不能忽略它。可用下面命令观察上下文和各层是否复用：

```bash
docker build --progress=plain -t demo/order-service:local .
```

连续构建两次，第二次再仅修改 `src/` 中一个 Java 文件。重点看依赖预热步骤是否被复用，而不是只看总耗时。若 `pom.xml` 没变但依赖层仍反复执行，优先检查构建上下文、Dockerfile 前置指令，以及 CI 是否每次都在无缓存的独立执行器上运行。

## 不要把“缓存”误当成“可重复构建”

缓存解决的是速度，不保证产物一致。依赖版本使用浮动范围、插件未固定、基础镜像使用可变标签，都会让相同提交在不同时间产生不同输入。工程上应分开处理：

- 用依赖锁定或依赖收敛策略控制 Java 依赖；
- 为构建镜像和运行镜像选择团队认可的明确标签或摘要；
- 在 CI 记录镜像标识、Git 提交和构建命令；
- 定期做一次无缓存构建，避免长期缓存掩盖依赖或网络问题。

## 提交前检查清单

上线前，至少回答四个问题：源码小改是否避免重新下载依赖？最终镜像中是否没有 Maven 和源码？CI 的缓存是否跨任务可用且可失效？禁用缓存后是否仍能完整构建？

把这些问题变成流水线日志或镜像检查项，比在故障时临时清空缓存更可靠。缓存层次清晰后，构建速度、镜像边界和问题定位会一起变得可控。

## 参考

- Docker Docs: [Optimize cache usage in builds](https://docs.docker.com/build/cache/optimize)
- Docker Docs: [Multi-stage builds](https://docs.docker.com/build/building/multi-stage)
- Docker Docs: [Building best practices](https://docs.docker.com/build/building/best-practices)
