---
title: 订单写入如何做到可重试：用幂等键守住接口边界
date: 2026-07-26T09:01:00+08:00
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

网络超时不等于服务端没有执行。客户端在收不到响应时重试，网关因短暂故障重放请求，消息消费者重复投递，都会让同一笔“创建订单”到达服务两次甚至更多次。若接口只按“收到一次请求就插入一行”实现，重复扣库存、重复发券会在流量恢复后集中出现。

解决重点不是让网络永不重试，而是让**同一业务操作重试任意次，最终只生效一次**。这就是写接口的幂等边界。本文以创建订单为例，给出一套在 Spring Boot 与关系型数据库中可落地的做法。

<!--more-->

## 先定义：什么才是同一次操作

不要用请求体全文或客户端时间戳判重。JSON 字段顺序、无关字段和时钟偏差都会让这类方案不稳定。更可靠的方式是由调用方为一次用户动作生成随机幂等键，并在重试时原样带上：

```http
POST /api/orders
Idempotency-Key: 8fdb9965-5b57-4270-a75d-7e19d390fcd8
Content-Type: application/json

{"skuId": 101,"quantity": 2}
```

服务端应把幂等键与调用方身份、业务类型绑定。例如 `(user_id, operation, idempotency_key)`；不能只以全局 key 为唯一条件，否则不同用户偶然使用相同值时会互相干扰。键应有足够随机性，且不要把手机号、订单号等可推测信息直接当作 key。

## 数据库唯一约束是最后防线

应用内先查询再插入并不可靠：两个并发请求都可能在查询时看不到记录，然后同时插入。必须让数据库唯一索引参与竞争。

```sql
create table idempotency_record (
    id bigint primary key generated always as identity,
    user_id bigint not null,
    operation varchar(64) not null,
    idempotency_key varchar(128) not null,
    request_hash varchar(64) not null,
    status varchar(16) not null,
    order_id bigint null,
    created_at timestamp not null,
    updated_at timestamp not null,
    unique (user_id, operation, idempotency_key)
);
```

`request_hash` 不是用来替代幂等键，而是用于检测调用方错误：同一个 key 却携带不同商品或数量时，不能悄悄返回第一次的结果，应返回明确的冲突错误，要求调用方生成新 key。哈希只记录规范化后的关键业务字段；不要把密码、令牌等敏感原文放入表中。

## 把“占位、执行业务、完成”放进一个状态机

记录至少应有 `PROCESSING`、`SUCCEEDED` 和 `FAILED` 三种状态。请求到达后，先尝试插入 `PROCESSING` 记录；插入成功的请求才拥有执行业务的资格。冲突请求读取已有记录：成功则返回同一个订单结果，处理中则提示稍后重试，失败则按业务规则决定是否允许用新 key 发起一次新操作。

下面的伪代码刻意省略了 Web 层细节，重点是事务边界：

```java
@Transactional
public CreateOrderResult createOrder(long userId, String key, CreateOrderCommand command) {
    String hash = command.businessHash();
    IdempotencyRecord record = recordRepository.tryInsertProcessing(userId, "CREATE_ORDER", key, hash);

    if (record == null) {
        record = recordRepository.findByUserIdAndOperationAndKey(userId, "CREATE_ORDER", key)
                .orElseThrow();
        if (!hash.equals(record.getRequestHash())) {
            throw new ConflictException("同一幂等键对应的请求内容不同");
        }
        if (record.isSucceeded()) {
            return new CreateOrderResult(record.getOrderId());
        }
        throw new RetryLaterException("请求正在处理");
    }

    Order order = orderService.create(userId, command);
    record.markSucceeded(order.getId());
    return new CreateOrderResult(order.getId());
}
```

这里的关键不是 ORM 方法名，而是 `tryInsertProcessing` 必须依赖前述唯一约束，并正确处理唯一键冲突。订单写入和将记录标为成功应处于同一个本地事务：事务回滚时，占位记录也不能留下一个错误的“成功”。

## 不要在事务里直接调用外部系统

创建订单后常常要发消息、扣积分或调用支付系统。若在数据库事务中直接发 HTTP 请求，远端已经成功但本地事务回滚时，两个系统就出现不一致；反过来，本地提交后进程崩溃又可能导致消息没有发出。

更稳妥的方式是使用 outbox：在同一个本地事务中写入订单、幂等结果和一条待发送事件；独立投递器再可靠地发送事件。消费者也必须按事件 ID 做幂等，不能因为生产端有幂等键就放弃防重。

## 为状态设置可观测性和过期策略

`PROCESSING` 不能永久存在。进程在事务外崩溃、锁等待或异常路径处理不当，都可能留下它。应记录创建时间，并对超过正常处理上限的记录告警；修复前先结合订单表、日志和事务状态确认是否已实际生效，不能由定时任务直接把所有旧记录改为失败。

建议至少监控以下指标：

- 幂等键命中次数，以及成功结果复用次数；
- 同 key、不同请求哈希的冲突次数；
- `PROCESSING` 状态的最长停留时间；
- 唯一键冲突和 outbox 投递失败次数。

幂等记录也需要保留期。保留时长应覆盖客户端最大重试窗口、异步任务重放窗口和审计需求；到期清理前，应确认旧 key 再次到达时是否可能造成业务风险。对支付、退款等高风险操作，通常还需要业务订单号或支付渠道流水号提供第二层约束。

## 上线前验证四个场景

不要只写一条正常路径测试。至少验证：同 key 串行重试返回同一订单；同 key 并发请求只创建一笔订单；同 key 不同请求体得到冲突；业务事务抛异常后不会留下成功结果。再在测试环境模拟客户端超时：让服务端提交订单后延迟响应，客户端重试应读回已创建的订单。

幂等不是一个注解，而是接口协议、唯一约束、事务边界和异步消费共同组成的闭环。把边界设计在写接口入口，才能把不可避免的重试变成可预测的正常流程。
