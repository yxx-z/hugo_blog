---
title: 排查 Java 数据库连接池耗尽：从症状到可复现的修复
date: 2026-07-23T09:01:12+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - 数据库
  - Spring Boot
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

线上看到“获取数据库连接超时”时，不要先把连接池最大连接数调大。它可能是连接没有归还、慢 SQL 长时间占用连接，或者应用并发已超过数据库能安全承受的范围。盲目扩容会把等待从应用连接池转移到数据库，最后让故障更难定位。

本文给出一条可操作的排查路径：先确认连接到底在做什么，再用最小改动修复泄漏或不合理的事务边界。

<!--more-->

## 先读懂三个计数

无论使用哪种连接池，诊断时都需要同时看三个量：

- **活跃连接**：正在被业务线程借出、尚未归还的连接数；
- **空闲连接**：池中可立即借出的连接数；
- **等待线程**：正在等待连接的请求数。

典型的危险信号是：活跃连接长期贴近上限、空闲连接为零、等待线程持续增加。此时不要只盯着错误日志中的超时异常；它只是最后一个拿不到连接的请求，不一定是最早占住连接的请求。

如果活跃连接周期性升高又快速回落，更像突发流量或短时间慢查询；如果它在流量回落后仍不下降，则优先怀疑连接泄漏、未结束事务，或某段代码在持有连接时做了远程调用。

## 先保留现场，而不是立刻重启

重启可以恢复服务，却会抹掉最有价值的线索。出现连接等待时，优先收集以下信息：

```bash
# 查看 JVM 中与数据库、线程池相关的线程；PID 换成实际进程号
jcmd <PID> Thread.print > /tmp/thread-dump.txt

# PostgreSQL 示例：查看当前会话与正在执行的语句
psql "$DATABASE_URL" -c '
select pid, usename, state, wait_event_type, wait_event,
       now() - query_start as running_for, left(query, 200) as query
from pg_stat_activity
where datname = current_database()
order by query_start nulls last;'
```

线程转储能回答“谁在等连接、谁可能持有连接”；数据库会话能回答“连接是在执行 SQL、等待锁，还是已经空闲但没有归还”。两边的时间点要尽量接近，否则容易把已结束的慢查询误判成当前问题。

生产环境执行诊断 SQL 前，应确认账号权限和查询范围。不要直接批量终止会话；先识别业务实例、事务状态和锁等待关系。

## 最常见的泄漏：资源没有放进 finally

JDBC 中 `Connection`、`PreparedStatement` 和 `ResultSet` 都必须关闭。不要依赖连接池的回收线程来“兜底”，它只能延后暴露问题，不能保证事务状态正确。

错误写法通常出现在异常分支提前返回：

```java
Connection connection = dataSource.getConnection();
PreparedStatement statement = connection.prepareStatement(sql);
if (!valid(input)) {
    return; // connection 没有归还
}
statement.executeUpdate();
connection.close();
```

用 try-with-resources 把归还动作与作用域绑定：

```java
public int updateStatus(DataSource dataSource, long orderId, String status)
        throws SQLException {
    String sql = "update orders set status = ? where id = ?";
    try (Connection connection = dataSource.getConnection();
         PreparedStatement statement = connection.prepareStatement(sql)) {
        statement.setString(1, status);
        statement.setLong(2, orderId);
        return statement.executeUpdate();
    }
}
```

退出 `try` 块时，连接池代理会把连接归还到池中，而不是物理关闭数据库连接。若手动管理事务，异常路径还要显式回滚；更推荐把事务交给 Spring 管理，避免每个分支都手写提交与回滚。

## 避免把远程调用包在事务里

连接未泄漏不代表使用合理。下面的代码会在 HTTP 调用期间持续占有数据库连接：

```java
@Transactional
public void createOrder(CreateOrderCommand command) {
    orderRepository.save(Order.create(command));
    paymentClient.createPayment(command.orderId());
}
```

支付服务抖动时，订单事务和连接都会被拖长；并发一高，连接池先耗尽，健康请求也会受影响。更稳妥的边界是：事务内只完成本地状态变更，提交后再异步或通过可靠消息触发外部动作。若业务必须同步调用，也应将远程调用移到事务外，并明确失败补偿策略。

```java
public void createOrder(CreateOrderCommand command) {
    Long orderId = transactionTemplate.execute(status ->
            orderRepository.save(Order.create(command)).getId());
    paymentClient.createPayment(orderId);
}
```

这并不自动解决分布式一致性问题，但至少不会让不可控的网络等待长期占住数据库连接。涉及扣款、库存等关键链路时，还需要幂等键、状态机和补偿任务配合。

## 用超时和告警缩短发现时间

连接池的等待超时应小于接口整体超时，并与网关、HTTP 客户端的超时预算协调。这样连接池拥塞能尽早失败和降级，而不是让请求一直排队。

同时至少监控：活跃连接比例、等待连接数、获取连接耗时、慢 SQL 数量、长事务数量。告警条件不要只设“连接数达到上限”；连续一段时间的高活跃比例加上等待线程增长，通常更早、更可靠。

修复后要做一次压测或集成测试：让数据库故意变慢、让下游调用超时，确认活跃连接能够在请求结束后回落。只有连接曲线恢复，而不是仅仅错误日志消失，才说明问题真正关闭了。
