---
title: Spring Boot 线程池隔离：别让一个慢依赖拖住全部请求
date: 2026-07-28T09:00:42+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Spring Boot
  - 分布式系统
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

订单接口里既有“创建订单”这样的核心路径，也可能要同步调用推荐、画像、积分等非核心服务。若它们共用 Web 请求线程或同一个业务线程池，一个慢依赖就会逐步耗尽全部工作线程，最后连不依赖它的请求也排队超时。

解决重点不是创建更多线程，而是按**故障边界**做线程池隔离：不同重要性、不同稳定性的工作，使用彼此独立且容量受限的执行器。

<!--more-->

## 先识别应该隔离的工作

适合单独隔离的通常是以下几类：

- 访问网络依赖：HTTP、RPC、对象存储、消息管理接口；
- 可降级的附加能力：推荐、埋点、通知、异步补偿；
- 耗时不稳定的本地任务：报表生成、文件转换、批量计算；
- 需要严格保护的核心写路径：下单、扣款、库存确认。

不要按“每个方法一个线程池”机械拆分。池子过多会让容量难以估算，也会掩盖依赖过慢的问题。先按调用链和业务优先级划分，例如 `orderCoreExecutor`、`profileExecutor`、`reportExecutor`，并明确每个池子的拒绝后应该返回什么结果。

## 配置有界队列和明确的拒绝策略

`ThreadPoolTaskExecutor` 默认配置不适合作为生产容量模型。关键参数必须显式设置，尤其是队列容量和拒绝策略。下面的画像查询池在压力下会快速拒绝新任务，而不是无限堆积等待：

```java
@Configuration
@EnableAsync
public class ExecutorConfig {

    @Bean("profileExecutor")
    public Executor profileExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(100);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("profile-");
        executor.setRejectedExecutionHandler(
                new ThreadPoolExecutor.AbortPolicy());
        executor.initialize();
        return executor;
    }
}
```

`AbortPolicy` 会抛出 `RejectedExecutionException`，调用方必须捕获它并执行降级。不要改用无界队列来“消除异常”：队列只会把拒绝延后，最终表现为延迟持续上升、内存压力和更难恢复的超时。

线程数也不能凭 CPU 核数直接套公式。I/O 等待型任务可比 CPU 型任务使用更多线程，但上限仍受下游连接池、下游限流和本机内存约束。先从保守值开始，基于压测和线上指标调整。

## 异步不等于自动隔离

使用 `@Async` 时，必须指定执行器；否则任务可能落到默认执行器，隔离边界就消失了。

```java
@Service
public class ProfileService {

    @Async("profileExecutor")
    public CompletableFuture<Profile> loadProfile(Long userId) {
        Profile profile = remoteClient.query(userId);
        return CompletableFuture.completedFuture(profile);
    }
}
```

调用端也不能无限等待异步结果。给 `CompletableFuture` 设置小于接口剩余时间的等待上限，并在超时、异常或任务被拒绝时返回默认画像。对于下单等核心链路，推荐把画像查询移到主流程之外，避免“异步后又立即 `join()`”这种表面异步、实际阻塞的写法。

## 把拒绝视为可观测的业务事件

线程池隔离是否生效，不能只看 CPU。至少应采集每个执行器的：

- 活跃线程数、当前池大小和队列长度；
- 已完成任务数与任务执行耗时；
- 拒绝次数；
- 下游调用的成功率、超时率和延迟分位数。

当某个非核心池持续满载时，正确动作通常是降级、限流或修复慢依赖，而不是立刻调大队列。若核心池也出现排队，应回到入口限流、数据库连接池和慢查询等环节排查。

## 发布前的验证清单

压测时可以人为让画像服务延迟数秒，然后确认：核心下单接口的延迟没有随之恶化；画像任务达到队列上限后出现可统计的拒绝；降级响应符合产品预期；请求结束后线程数和队列长度能恢复到稳定水平。

线程池隔离不是替代超时、重试和熔断的万能开关。它的价值在于把资源上限和失败范围写进系统结构：慢依赖可以失败，但不应获得拖垮整个服务的机会。
