---
title: Java 服务优雅停机：先停止接流量，再排空请求
date: 2026-07-25T09:00:48+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Spring Boot
  - Kubernetes
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

发布 Java 服务时，最容易被忽略的一步不是启动新实例，而是怎样停止旧实例。若进程收到终止信号后立刻退出，正在执行的订单、支付回调或文件上传会被中断；客户端重试后，还可能把一次操作放大成重复操作。

优雅停机的目标不是“让进程晚一点退出”，而是在有限时间内完成一套明确动作：**停止接收新流量、让已进入的请求完成、关闭后台工作、最后退出进程**。这套顺序需要应用、负载均衡和编排平台一起配合。

<!--more-->

## 先区分两类请求

一个 HTTP 服务停机时，通常同时存在两类工作：

- **新请求**：不应该再进入即将下线的实例；
- **进行中的请求**：应在合理的截止时间内继续执行，并释放数据库连接、线程和文件句柄。

只依赖进程信号无法区分它们。正确的入口是健康检查：实例进入停机状态后，readiness 应立即变为不可用。负载均衡器或 Kubernetes 随后会把它从服务端点中摘除，新连接自然转到其他健康实例。

注意：readiness 变为失败不等于流量已经消失。代理、DNS、长连接和客户端连接池都有传播延迟，因此应用还要保留一段排空窗口。

## Spring Boot 的基础配置

Spring Boot 可以在关闭 Web 容器时等待活动请求完成。下面是一份基础配置：

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

`server.shutdown: graceful` 的意义是先拒绝新的 Web 请求，再等待已开始处理的请求结束；`timeout-per-shutdown-phase` 则限制每个关闭阶段最多等待多久。超时并不表示业务一定安全完成，只表示应用不能无限期卡在退出流程中。

配置完成后，仍要检查业务代码是否会无期限阻塞。例如没有读取超时的 HTTP 调用、无超时的锁等待，都会让排空阶段失效。每个出站调用都应有自己的 deadline，并且小于服务的总停机窗口。

## Kubernetes 中的停机顺序

Kubernetes 删除 Pod 时会向容器发送终止信号，并在宽限期结束后强制终止。应用的排空时间必须小于该宽限期，并为网络摘流和进程收尾留下余量：

```yaml
spec:
  terminationGracePeriodSeconds: 40
  containers:
    - name: app
      image: example/app:latest
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        periodSeconds: 5
```

如果应用最多等待 25 秒，可以把总宽限期设为 40 秒，而不是恰好 25 秒。额外时间用于终止信号传递、端点更新、日志落盘和 JVM 退出。具体数值要根据最长正常请求、代理摘流延迟和实际演练结果确定。

对于长轮询、SSE 或 WebSocket，不能简单等到自然结束。需要在应用协议中设计关闭通知或连接最大存活时间；否则少量长连接就能持续占用整个停机窗口。

## 后台任务不能跟着“突然消失”

HTTP 请求排空后，定时任务、消息消费者和异步线程仍可能在运行。停机时应先停止拉取新消息，再等待已取得的消息处理完成；处理超时后交给消息系统重投，而不是在内存中无限等待。

可以为自己的后台组件实现关闭回调，并保证其幂等：

```java
import org.springframework.context.SmartLifecycle;

public final class WorkerLifecycle implements SmartLifecycle {
    private volatile boolean running;

    @Override
    public void start() {
        running = true;
    }

    @Override
    public void stop() {
        running = false; // 先停止领取新任务，再由工作线程完成当前任务
    }

    @Override
    public boolean isRunning() {
        return running;
    }
}
```

示例中的 `running` 只是控制入口。真正的消费者还需要等待当前任务、提交确认状态，并在超时后可恢复。不要在关闭回调里执行不可控的网络请求，也不要依赖 `Thread.stop()` 这类强制中断手段修复一致性。

## 用一次演练验证，而不是相信配置

至少在预发布环境做一次可重复的停机演练：

1. 发起一个耗时但正常的请求，并记录请求 ID；
2. 对实例执行滚动发布或发送终止信号；
3. 确认新请求不再被路由到该实例；
4. 确认进行中的请求在预算内成功或按约定失败；
5. 检查消息是否重复消费、数据库连接是否归还、进程是否在宽限期内退出。

同时记录停机开始时间、活动请求数、拒绝的新请求数、后台任务数和最终退出原因。只有这些数据能回答：是摘流太慢、请求本身太长，还是关闭阶段超时。

## 结语

优雅停机本质上是一份退出协议：健康检查负责停止接流量，Web 容器负责排空请求，后台组件负责停止领取新工作，平台的宽限期负责兜底。把每一段时间都设成可观察、可演练的预算，才能避免一次正常发布演变为用户可见的失败。
