---
title: 业务已提交但消息没发出：Java 服务的事务外盒实践
date: 2026-07-30T09:01:25+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Spring Boot
  - 分布式系统
  - 消息队列
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

下单成功后要通知库存、积分和物流。很多服务会先写订单，再调用消息队列；进程若恰好在数据库提交后崩溃，订单存在，事件却永远没有发出。反过来，先发消息再提交数据库，消费者可能先看到一笔最终会回滚的订单。

这不是把 `send()` 换成异步调用能解决的问题，而是数据库提交与消息投递属于两个独立操作。事务外盒（transactional outbox）用一个简单约束缩小这个不一致窗口：**业务数据和待投递事件必须在同一个本地事务中落库；真正发送由事务外的投递器负责。**

<!--more-->

## 先承认边界：不能承诺“恰好一次”

外盒能保证已提交的业务变更有一条可恢复的待发事件，但投递器在“消息已被 broker 接收、尚未来得及把事件标记为已发送”时崩溃，重启后仍会再次发送。因此生产端通常提供的是至少一次投递。

接受重复，才能把问题变得可控。消费者必须以事件 ID 防重，业务表也应保留自己的唯一约束。不要把“队列支持去重”当成唯一防线，因为重放、迁移和人工补发都可能绕过它。

## 在同一事务中写业务与事件

外盒表不需要复杂，但字段必须能支撑定位、重试和演进：

```sql
create table outbox_event (
    id varchar(36) primary key,
    aggregate_type varchar(64) not null,
    aggregate_id varchar(64) not null,
    event_type varchar(128) not null,
    payload text not null,
    status varchar(16) not null,
    retry_count integer not null default 0,
    next_attempt_at timestamp not null,
    created_at timestamp not null,
    sent_at timestamp null
);

create index idx_outbox_pending
    on outbox_event (status, next_attempt_at, created_at);
```

`payload` 应是可独立消费的事件事实，例如订单 ID、用户 ID、金额和事件版本；不要只传一个需要消费者再回调生产服务的临时对象。敏感字段同样不应进入事件。若确实要传个人信息，先按数据分类、脱敏和保留期要求设计。

在 Spring 服务中，订单与事件一同保存。只要事务回滚，两者都会消失；提交成功，两者都会存在：

```java
@Transactional
public Long createOrder(CreateOrderCommand command) {
    Order order = orderRepository.save(Order.create(command));

    OutboxEvent event = OutboxEvent.pending(
            UUID.randomUUID().toString(),
            "Order",
            order.getId().toString(),
            "order.created.v1",
            objectMapper.writeValueAsString(new OrderCreated(order.getId(), command.userId())));
    outboxRepository.save(event);

    return order.getId();
}
```

这里不应在事务方法内直接调用 broker。即使网络调用成功，也无法让远端操作跟随本地数据库回滚；网络慢还会拉长数据库事务，增加连接与锁的占用。

## 投递器要能并发运行和安全重试

投递器可由定时任务、独立进程或变更数据捕获组件实现。使用轮询时，多个实例必须只领取各自的事件。不同数据库对锁语法和隔离级别的支持不同，先在目标数据库做并发验证；一种常见思路是短事务内领取记录，再在事务外发送：

```java
public void dispatchOnce() {
    List<OutboxEvent> events = transactionTemplate.execute(status ->
            outboxRepository.claimPending(100, Instant.now()));

    for (OutboxEvent event : events) {
        try {
            messagePublisher.publish(event.getEventType(), event.getId(), event.getPayload());
            outboxRepository.markSent(event.getId(), Instant.now());
        } catch (Exception ex) {
            outboxRepository.reschedule(event.getId(), nextRetryAt(event));
        }
    }
}
```

`claimPending` 的目标是原子地把符合条件的记录改为 `SENDING`，并返回本实例拥有的事件。`SENDING` 还要有租约或超时恢复机制：实例异常退出后，超过租约的记录可重新变为待发送。重试间隔应逐步拉长，并设置最大次数；超过阈值的事件进入失败状态并告警，而不是无声地无限重试。

发送成功后的 `markSent` 可以是一次普通更新。它失败会导致重复发送，所以消费者防重仍然不可省略。

## 消费者用事件 ID 形成第二道边界

消费者处理前先尝试写入已处理记录，并依赖唯一键竞争。插入成功者执行副作用，冲突者说明该事件已处理过：

```sql
create table consumed_event (
    event_id varchar(36) primary key,
    consumer varchar(64) not null,
    consumed_at timestamp not null
);
```

已处理记录、消费业务更新和消息确认要放在同一个消费侧事务边界内。若业务更新失败，已处理标记也必须回滚，消息才能重试。记录的保留期要覆盖 broker 的最大重放周期和运维补发窗口；清理前需确认旧事件重新出现不会产生副作用。

## 上线前验证四个故障点

先在测试环境主动制造失败，而不是只验证一条成功消息：

- 订单事务回滚时，确认没有外盒事件；
- 数据库提交后停止应用，重启后确认事件最终投递；
- broker 已接收后让 `markSent` 失败，确认消费者只生效一次；
- 停止一个投递实例，确认它领取但未处理完成的事件可被其他实例恢复。

同时监控待发送事件数量、最老事件年龄、投递延迟、重试次数和失败事件数。外盒不是替代消息队列，而是把“提交后丢消息”从不可见偶发事故，变成数据库里可观测、可重试、可审计的工作项。先把投递做成至少一次，再在消费者收紧幂等边界，系统才能在故障发生时仍保持业务结果正确。
