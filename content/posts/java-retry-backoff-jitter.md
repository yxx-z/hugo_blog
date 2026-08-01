---
title: Java 调用外部服务重试：退避、抖动与幂等边界
date: 2026-08-01T09:02:25+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Spring Boot
  - 分布式系统
  - 工程实践
  - 稳定性
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

一次 HTTP 调用失败，不等于目标服务永久不可用；但无条件地立刻重试，往往会把一次短暂抖动放大成级联故障。真正要设计的不是“失败后再调一次”，而是明确：**什么请求可以重试、哪些错误值得重试、最多占用多少时间，以及重试流量如何不压垮下游。**

本文用 Java 服务间调用为例，给出一套不依赖特定框架的落地边界。示例以 `java.net.http.HttpClient` 编写，使用 Spring Boot、Feign 或 WebClient 时，判断逻辑同样适用。

<!--more-->

## 先判断：这次操作能否安全重复

重试会重复执行一次操作。查询、读取配置、按资源 ID 获取详情通常是幂等的；创建订单、扣款、发送短信则未必。不能因为接口是 `POST` 就一律不重试，也不能因为它是 `GET` 就忽略副作用。

对写操作，更可靠的做法是由调用方生成幂等键，并在每一次尝试中带上相同的键：

```http
POST /payments
Idempotency-Key: 6b35c737-2fc2-4b79-a63e-5e8e8e9cf8f1
Content-Type: application/json
```

服务端必须把幂等键与业务结果建立唯一关联：首次请求执行业务并保存结果；后续相同键返回同一结果，而不是再次扣款。只有服务端具备这个契约，客户端才可以对“请求可能已到达、但响应丢失”的场景进行重试。

不要把随机生成的键放在重试循环里。那会让每次请求都变成新操作，等于主动绕过幂等保护。

## 只重试短暂故障

重试策略应区分失败类型，而不是仅看是否抛异常：

- 连接超时、连接被重置、网关短暂不可达：通常可以重试。
- HTTP `429`、`502`、`503`、`504`：可在总时间预算内谨慎重试；若有 `Retry-After`，应优先遵守它。
- HTTP `400`、`401`、`403`、`404`、`422`：请求本身或权限有问题，重试没有价值。
- HTTP `500`：不能一概而论。若接口已声明幂等，且确认是临时服务错误，可以有限重试；否则保留错误上下文并交给人工或异步补偿。

同时要限制重试次数。一次用户请求内做两次额外尝试，已经会让下游瞬时负载接近三倍；下游正在过载时，更多重试通常只会延长恢复时间。

## 指数退避还不够：加入抖动

固定间隔，例如每 200 ms 重试，容易让同一时刻失败的大量请求在同一时刻再次到达下游。指数退避让等待时间逐步增长，抖动（jitter）则把重试打散。

下面的函数使用“全抖动”：第 `attempt` 次失败后的等待时间，随机落在 `0` 到当前上限之间。上限也会封顶，避免等待无限增长。

```java
import java.time.Duration;
import java.util.concurrent.ThreadLocalRandom;

static Duration nextDelay(int attempt) {
    long baseMs = 100;
    long maxMs = 1_000;
    long cap = Math.min(maxMs, baseMs * (1L << attempt));
    return Duration.ofMillis(ThreadLocalRandom.current().nextLong(cap + 1));
}
```

生产代码还要处理移位溢出，因此不要让 `attempt` 无上限增长。最简单的办法是把尝试次数固定为很小的值，例如首次调用加两次重试。

## 把总时间预算放在最外层

常见错误是只配置“单次连接超时”和“单次读取超时”。三次尝试加上两段退避，实际耗时可能远超上游允许的请求时限，最终线程白白占着连接和工作线程。

调用链应先确定总预算。例如入口还剩 800 ms 时，外部调用不应分配三个 500 ms 的尝试。下面的示例在每次发起请求前检查剩余时间，并把单次请求超时收敛到剩余预算内：

```java
static <T> T retry(CheckedSupplier<T> action, Duration budget) throws Exception {
    long deadline = System.nanoTime() + budget.toNanos();
    Exception last = null;

    for (int attempt = 0; attempt < 3; attempt++) {
        if (System.nanoTime() >= deadline) break;
        try {
            return action.get();
        } catch (Exception e) {
            last = e;
            if (attempt == 2) break;
            long leftNanos = deadline - System.nanoTime();
            long sleepMs = Math.min(nextDelay(attempt).toMillis(),
                    Math.max(0, leftNanos / 1_000_000));
            if (sleepMs <= 0) break;
            Thread.sleep(sleepMs);
        }
    }
    throw last == null ? new IllegalStateException("retry budget exhausted") : last;
}

@FunctionalInterface
interface CheckedSupplier<T> {
    T get() throws Exception;
}
```

示例为了突出边界直接使用 `Thread.sleep`。在 WebFlux 等非阻塞链路中，应使用框架的异步延迟机制，不能在事件循环线程阻塞等待。

## 重试必须和限流、熔断配合

重试解决的是偶发失败，不是持续故障。连续失败达到阈值后，熔断器应暂时拒绝新的实际调用，快速返回可识别的降级结果；半开状态只放少量探测请求。对于高并发入口，还应限制某个下游的并发数或排队长度，避免等待中的请求耗尽整个应用线程池。

日志和指标至少记录：下游名称、结果分类、尝试次数、累计耗时、是否耗尽预算。不要在每次可预期重试时打印完整异常栈；高峰期这会制造噪声并增加日志成本。对最终失败记录一次带关联 ID 的错误，再以计数器和耗时直方图观察趋势即可。

## 上线前检查清单

1. 写操作是否有服务端可验证的幂等键，而非仅靠客户端猜测？
2. 是否只对明确的短暂错误重试，并排除了参数、鉴权等确定性错误？
3. 每次尝试、退避等待和整个调用链是否共用一个总时间预算？
4. 是否设置了很小且可解释的最大尝试次数，并加入随机抖动？
5. 下游持续异常时，是否有并发隔离、熔断或降级，而不是无限排队？
6. 是否能从指标中区分首次成功、重试成功和最终失败？

重试不是可靠性的默认开关。先建立幂等契约，再把次数、时间和并发控制在边界内，短暂网络问题才不会演变成对下游的第二次冲击。
