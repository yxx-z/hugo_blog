---
title: Kafka 重复消费怎么收敛：Spring Boot 手动确认与幂等消费实践
date: 2026-08-02T09:00:38+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Spring Boot
  - Kafka
  - 消息队列
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

Kafka 消费端最危险的误解是“消息只会来一次”。消费者完成业务写入后、提交位移前宕机，重启后仍会收到同一条消息；位移提交成功但业务事务失败，则消息可能被跳过。前者是重复，后者是丢失。二者不能靠一个开关同时消除。

实际目标应改成：**允许消息至少一次到达，但让同一业务事件最多生效一次。**手动确认只负责缩小确认时机，幂等约束才是业务正确性的最后防线。

<!--more-->

## 先把确认放在业务成功之后

自动提交位移时，框架可能在业务逻辑尚未完成时推进消费进度。对会写数据库、调用下游或发起扣减的消息，应由监听器在本地处理成功后显式确认。

下面的配置让监听器自行决定确认时机：

```java
@Configuration
class KafkaConsumerConfig {

    @Bean
    ConcurrentKafkaListenerContainerFactory<String, OrderPaidEvent> kafkaListenerContainerFactory(
            ConsumerFactory<String, OrderPaidEvent> consumerFactory) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, OrderPaidEvent>();
        factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        return factory;
    }
}
```

业务成功后再调用 `acknowledge()`：

```java
@KafkaListener(topics = "order-paid", groupId = "inventory-service")
public void onOrderPaid(OrderPaidEvent event, Acknowledgment acknowledgment) {
    inventoryService.reserve(event);
    acknowledgment.acknowledge();
}
```

这段代码的顺序不能颠倒。确认在前，进程随后崩溃，消息不会自动重放；确认在后，崩溃时会重放。因此即使使用手动确认，`reserve` 也必须能承受重复调用。

## 用事件 ID 做消费侧去重

不要仅按订单号去重。一个订单可以有支付成功、取消、退款等不同事件，正确的去重键通常是生产者生成且跨重试不变的 `eventId`。为每个消费者维护已处理表，并让数据库唯一键裁决并发竞争：

```sql
create table consumed_event (
    consumer_name varchar(64) not null,
    event_id varchar(64) not null,
    consumed_at timestamp not null,
    primary key (consumer_name, event_id)
);
```

库存预占和已消费标记必须在同一个本地事务内完成：

```java
@Transactional
public void reserve(OrderPaidEvent event) {
    try {
        jdbcTemplate.update("""
                insert into consumed_event (consumer_name, event_id, consumed_at)
                values (?, ?, current_timestamp)
                """, "inventory-service", event.eventId());
    } catch (DuplicateKeyException duplicate) {
        return;
    }

    int changed = jdbcTemplate.update("""
            update inventory
               set available = available - ?
             where sku_id = ? and available >= ?
            """, event.quantity(), event.skuId(), event.quantity());
    if (changed != 1) {
        throw new InsufficientInventoryException(event.skuId());
    }
}
```

首次处理时，插入去重记录和扣减库存一起提交；任何一步抛异常，两者一起回滚，监听器不确认位移，后续可以重试。重复到达时，唯一键冲突后直接返回，随后确认位移，不会再次扣减。

注意：不能把 `DuplicateKeyException` 抛出事务方法。如果异常没有被处理，事务会标记为回滚，监听器会反复收到同一条已处理消息。对复杂场景，更稳妥的方式是用数据库方言支持的“冲突即忽略”语句，或将去重插入封装为能明确返回插入结果的仓储方法。

## 失败要区分可重试与不可重试

数据库暂时不可用、下游超时等短暂错误可以抛出，让消息稍后重试；库存不足、事件格式不合法、找不到 SKU 等确定性业务错误，不能无限重试。否则一个坏消息会长期阻塞分区后续消息。

对不可重试错误，应记录主题、分区、位移、`eventId` 和原因，并投递到死信主题或进入人工处理队列。不要只记录反序列化后的对象；原始消息和 headers 往往是排查事件版本不兼容的关键。死信处理完成后，若要重放，仍应保留同一个 `eventId`，不要重新生成。

## 位移、数据库与外部副作用的边界

手动确认和本地事务不能把 Kafka 位移、数据库、第三方 HTTP 调用变成一个全局原子事务。尤其是“已调用支付/短信接口，但进程在确认前退出”时，重放仍可能再次调用外部系统。

处理这类副作用时至少补上一层约束：

- 对外部写操作传递稳定的幂等键，通常直接使用 `eventId`。
- 数据库状态变更与待发送事件使用事务外盒保存，再由独立投递器发送。
- 监听器并发数不要超过目标数据库和下游的可承受能力；先按分区数和连接池容量测算。

前两层解决“重复到达”和“调用结果不确定”，最后一层避免消费堆积时把故障扩大成连接池耗尽。

## 上线前验证

不要只用一条成功消息验证。至少演练以下四种情况：

1. 数据库事务提交后、位移确认前终止进程，确认重放后库存只扣减一次。
2. 收到相同 `eventId` 的并发消息，确认唯一键只允许一个处理者进入业务更新。
3. 让数据库更新失败，确认去重记录没有残留，消息能够在恢复后处理。
4. 注入格式错误或库存不足事件，确认它进入可追踪的失败路径，且不无限阻塞后续消息。

同时监控消费滞后、重复命中数、失败重试次数、死信数量和最老未处理消息时间。Kafka 的重复投递不是异常分支，而是必须纳入设计的常规路径：确认放在成功之后，副作用放在幂等边界内，故障恢复时业务结果才可预测。
