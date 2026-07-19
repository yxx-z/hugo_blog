---
title: 别只配一个超时：Java 服务的超时预算实践
date: 2026-07-19T09:01:00+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - 分布式系统
  - 工程实践
  - 可观测性
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

线上接口“偶发很慢”时，很多人的第一反应是把 HTTP 客户端超时从 1 秒调到 5 秒。这通常只是在把故障暴露得更晚：上游请求已经没有耐心了，下游却还在占用线程、连接和数据库资源。

更可靠的做法是把一次请求允许消耗的时间，当成一份必须分配的**超时预算**。每层调用只能使用剩余预算，而不是各自独立地等待一个固定时长。

<!--more-->

## 为什么单个超时会失控

假设一个网关给订单接口 2 秒，订单服务依次调用库存、优惠和用户服务。若三个客户端都配置 2 秒超时，最坏情况下订单服务可能等待 6 秒；网关早已返回失败，但后端工作仍在继续。

这会造成三个问题：

- 取消不了的请求继续占用 Tomcat 工作线程；
- 连接池中的连接迟迟不归还，后续正常请求开始排队；
- 慢依赖发生抖动时，重试又放大了它的负载。

因此，超时不是“给下游多久”，而是“在调用方还来得及完成响应前，下游最多能占用多久”。

## 先定义一份预算

以接口目标延迟 800ms 为例，不应把 800ms 全交给下游。需要预留应用自身的鉴权、序列化、日志、调度抖动和返回响应时间。

可以先采用一份保守预算：

```text
总预算                 800 ms
应用本地处理与预留       180 ms
库存服务                220 ms
优惠服务                180 ms
用户服务                140 ms
兜底余量                 80 ms
```

这里的数字不是通用标准。它应来自接口延迟分布和业务优先级：关键路径优先，非关键数据在预算不足时降级或省略。预算总和必须小于调用方对外承诺的时间。

## 在入口记录 deadline

推荐在入口把截止时间写入请求上下文，内部调用每次计算剩余时间。不要把“还剩多少毫秒”直接透传，因为排队和网络传输会让这个值过期。

下面示例使用 Java 标准库表达这个思路：

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public final class DownstreamClient {
    private final HttpClient client = HttpClient.newBuilder()
            .connectTimeout(Duration.ofMillis(100))
            .build();

    public String get(URI uri, long deadlineNanos) throws Exception {
        long leftNanos = deadlineNanos - System.nanoTime();
        long leftMillis = Duration.ofNanos(leftNanos).toMillis();
        if (leftMillis <= 0) {
            throw new IllegalStateException("request deadline exceeded");
        }

        // 给响应组装留出余量，避免刚收到下游响应就已超时。
        long timeoutMillis = Math.max(1, leftMillis - 30);
        HttpRequest request = HttpRequest.newBuilder(uri)
                .timeout(Duration.ofMillis(timeoutMillis))
                .GET()
                .build();
        return client.send(request, HttpResponse.BodyHandlers.ofString()).body();
    }
}
```

入口可以用 `System.nanoTime() + Duration.ofMillis(800).toNanos()` 创建 `deadlineNanos`。`nanoTime` 适合计算耗时；不要用可能被校时调整的墙上时钟来判断剩余时间。

## 连接超时、读取超时与业务 deadline 要分开

这三个概念经常被混用：

1. **连接超时**：TCP 或 TLS 建连最多等待多久。它通常应较短；
2. **请求超时**：一次 HTTP 请求从发起到完成最多多久；
3. **业务 deadline**：整条调用链最终必须结束的时间点。

客户端请求超时应不大于业务剩余预算。连接超时则应更小，否则一次连不上就吞掉了大半预算。数据库、消息队列和缓存客户端也应遵循同一原则，不能只给 HTTP 设置超时。

## 重试必须从预算里扣除

重试不是免费的可用性。只有同时满足以下条件才考虑重试：操作幂等、失败类型可判定为瞬时、剩余预算足够完成下一次尝试。

例如库存查询可以在 400ms 总预算里尝试两次：第一次 180ms，退避 20ms，第二次最多 180ms；如果只剩 100ms，就直接降级。创建订单这类非幂等写操作，不能因为超时就盲目重发，否则可能产生重复订单。

重试次数、退避时间和剩余预算应写入指标。只看“重试后成功率”会掩盖依赖变慢的事实。

## 用指标验证配置是否真的有效

至少按下游名称统计：请求耗时分位数、超时数、连接池等待时间、重试次数和降级次数。日志中记录 `deadline_ms`、`remaining_ms` 与失败原因，才能区分“下游慢”“本地排队”还是“预算本身过小”。

上线时先观察超时是否集中在某一个依赖，并比较变更前后的线程池活跃数与连接池等待。若缩短超时后错误率上升，不应立即把数值调大；先确认是否缺少缓存、限流、隔离或降级。

## 结语

超时预算的目标不是让更多请求等待成功，而是让系统在依赖异常时及时止损。把 deadline 从入口传到每个依赖，把重试也纳入预算，再用指标校准数值，才能避免一次慢调用拖垮整条链路。
