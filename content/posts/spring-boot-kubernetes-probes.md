---
title: Spring Boot 上 Kubernetes 探针别共用一个健康检查
date: 2026-07-22T09:00:47+08:00
draft: false
author:
  name: yxx
tags:
  - Spring Boot
  - Kubernetes
  - 可观测性
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

把 `/actuator/health` 同时配置给 Kubernetes 的启动、存活和就绪探针，看起来省事，却容易把不同故障混为一谈：数据库短暂抖动可能触发重启，应用刚启动又可能被过早摘流量或杀掉。

更稳妥的思路是先定义三个问题，再让探针只回答它该回答的问题：**进程是否卡死、实例能否接流量、应用是否已完成启动**。Spring Boot Actuator 已为 Kubernetes 提供 liveness 与 readiness 健康组；Kubernetes 也明确区分了三类 probe。关键不在于端点数量，而在于每个端点的依赖边界。

<!--more-->

## 先区分三种探针的动作

- **startupProbe**：容器启动阶段是否已经可用。它未成功前，Kubernetes 不执行 liveness 和 readiness。适合启动慢、需要预热或执行迁移的服务。
- **livenessProbe**：进程是否已经进入无法自行恢复的状态。连续失败会导致容器被重启。
- **readinessProbe**：实例当前能否接收业务流量。失败时实例会从 Service 的可用端点中移除，但容器不会因此重启。

因此，“下游 Redis 暂时不可用”通常是就绪问题，不天然是存活问题。把外部依赖放进 liveness，故障时可能形成重启风暴：实例重启后仍连不上依赖，反而放大恢复时间。

## Spring Boot 的端点与暴露方式

引入 `spring-boot-starter-actuator` 后，可启用并映射探针健康组：

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,info
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

在常见的 Web 应用中，可访问：

```text
/actuator/health/liveness
/actuator/health/readiness
```

先在本地或测试环境执行验证，确认 HTTP 状态和实际健康组内容，而不是假设配置已生效：

```bash
curl -i http://127.0.0.1:8080/actuator/health/liveness
curl -i http://127.0.0.1:8080/actuator/health/readiness
```

若管理端口与业务端口分离，还应决定探针请求哪个端口。探针走独立管理端口时，业务 HTTP 线程、连接器或网络路径异常未必会反映出来；走业务端口更贴近真实流量，但要控制暴露范围。没有放之四海皆准的选择，关键是把选择写进部署约定并测试。

## 一份可起步的 Deployment 配置

下面示例把三个探针明确指向各自端点。数值只是起点，必须按镜像冷启动时间和接口延迟调整：

```yaml
containers:
  - name: order-service
    image: registry.example.com/order-service:20260722
    ports:
      - containerPort: 8080
    startupProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      periodSeconds: 5
      failureThreshold: 24
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 2
```

这里启动探针最多允许约 120 秒（`periodSeconds × failureThreshold`）完成启动。超过这个窗口并不说明应用一定有 bug，也可能是阈值与真实启动耗时不匹配；应结合启动日志、镜像拉取、JVM 预热和初始化任务定位，而不是直接无限调大。

## 健康指标的边界比配置更重要

liveness 应尽量只反映应用自身是否还在正常推进，例如死锁、事件循环长期阻塞或内部状态不可恢复。不要把数据库、消息队列、第三方 HTTP 服务直接塞进 liveness。

readiness 可以更贴近“此刻能否服务”。但也不要机械要求所有非关键依赖都健康：推荐系统不可用时，订单创建仍能完成的服务可以降级推荐，而非把自身整体摘流量。应该从业务关键路径倒推：缺少什么能力时，这个实例就不应再接请求？

## 上线前做一次故障演练

配置提交前，至少验证四件事：

1. 人为让应用启动变慢，确认 startupProbe 期间不会被 liveness 提前重启；
2. 模拟非关键下游超时，确认实例按设计继续服务或降级；
3. 模拟关键依赖不可用，确认 readiness 失败后流量被摘除，而不是反复重启；
4. 查看 `kubectl describe pod` 的探针事件，并在监控中区分重启次数、未就绪副本数和下游失败率。

探针不是“有 200 响应就够了”的 YAML 模板。把存活、接流量和启动完成三个语义拆开，才能在依赖故障、发布预热和真实卡死时，让 Kubernetes 做出不同且可预期的动作。

## 参考

- Spring Boot Reference Documentation: [Endpoints / Kubernetes Probes](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html)
- Kubernetes Documentation: [Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/workloads/pods/probes)
