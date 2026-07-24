---
title: Redis 缓存击穿：用互斥重建保护热点键，而不是把数据库当兜底
date: 2026-07-24T09:01:35+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Redis
  - Spring Boot
  - 缓存
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

某个热点商品详情刚好过期时，几百个并发请求会同时发现 Redis 未命中，并一起查询数据库。这不是普通的缓存未命中，而是**缓存击穿**：单个热点键失效，瞬时把读压力集中到下游。把缓存 TTL 调长只能降低概率，不能保证重建期间只有一个请求回源。

一个可控的起点是：缓存未命中后，只有拿到互斥锁的请求允许重建；其他请求短暂等待并再次读取缓存。锁的目标不是“绝不回源”，而是把同一键的并发回源收敛为有限次数。

<!--more-->

## 先确认是不是击穿

击穿常见于访问量高、数据可按主键读取、并且设置了固定过期时间的键。它与另外两类问题不同：

- **缓存穿透**：请求的是不存在的数据，缓存和数据库都没有；需要空值缓存、参数校验或布隆过滤器等措施。
- **缓存雪崩**：大量键在相近时间同时过期，或者缓存集群整体不可用；需要错峰过期、多级缓存和容量治理。
- **缓存击穿**：少量热点键过期，关键是避免同一个键被并发重建。

先用监控确认：某个 key 的未命中增多时，数据库中对应主键的查询 QPS、连接等待和接口延迟是否同步上升。没有这些证据，不要把所有慢查询都归因于缓存。

## 两次读缓存的互斥重建

下面示例使用 `StringRedisTemplate`。锁键必须带业务前缀，锁值使用随机 token；释放时只能删除自己持有的锁。`setIfAbsent` 同时设置过期时间，避免“先加锁、进程在设置过期前崩溃”留下永久锁。

```java
@Service
@RequiredArgsConstructor
public class ProductQueryService {
    private final StringRedisTemplate redis;
    private final ProductRepository productRepository;
    private final ObjectMapper objectMapper;

    public ProductView getProduct(long id) throws JsonProcessingException {
        String cacheKey = "product:detail:" + id;
        String cached = redis.opsForValue().get(cacheKey);
        if (cached != null) {
            return objectMapper.readValue(cached, ProductView.class);
        }

        String lockKey = "lock:product:detail:" + id;
        String token = UUID.randomUUID().toString();
        Boolean locked = redis.opsForValue().setIfAbsent(lockKey, token, Duration.ofSeconds(5));
        if (Boolean.TRUE.equals(locked)) {
            try {
                // 拿锁后必须再读一次；可能已有请求完成了缓存回填。
                cached = redis.opsForValue().get(cacheKey);
                if (cached != null) {
                    return objectMapper.readValue(cached, ProductView.class);
                }

                ProductView product = productRepository.findViewById(id);
                if (product == null) {
                    return null; // 不存在数据应按业务决定是否缓存空值
                }
                redis.opsForValue().set(cacheKey, objectMapper.writeValueAsString(product),
                        Duration.ofMinutes(10).plusSeconds(ThreadLocalRandom.current().nextLong(60)));
                return product;
            } finally {
                // 仅当锁值仍是自己的 token 才删除，不能直接 delete(lockKey)。
                String script = "if redis.call('get', KEYS[1]) == ARGV[1] then "
                        + "return redis.call('del', KEYS[1]) else return 0 end";
                redis.execute(new DefaultRedisScript<>(script, Long.class), List.of(lockKey), token);
            }
        }

        // 未获得锁：短暂等待后重读，不要立即大量重试。
        LockSupport.parkNanos(Duration.ofMillis(30).toNanos());
        cached = redis.opsForValue().get(cacheKey);
        if (cached != null) {
            return objectMapper.readValue(cached, ProductView.class);
        }
        throw new ResponseStatusException(HttpStatus.SERVICE_UNAVAILABLE,
                "热点数据正在重建，请稍后重试");
    }
}
```

代码中的 `DefaultRedisScript`、`List`、`UUID`、`Duration`、`LockSupport`、`ResponseStatusException` 和 `HttpStatus` 需要按项目补齐对应 import。这里让未拿到锁且二次读取仍失败的请求快速失败；对读接口也可以返回本地短缓存中的旧值，但不要无上限排队等待，否则保护缓存会变成堆积请求。

## 锁超时不是拍脑袋配置

锁 TTL 必须大于一次数据库查询与缓存写入的正常上界，并留出网络抖动余量；同时必须远小于客户端可接受的超时。若重建可能超过锁 TTL，旧持锁者完成后可能误删新持锁者的锁，所以 token 校验不能省略。更根本的改进是缩短回源路径，而不是无限拉长锁时间。

固定 TTL 还会让热点键在同一时刻集中失效。示例给过期时间增加了 0 到 59 秒的随机值，只是为了错峰；随机范围应结合数据允许陈旧时间确定，不能因为“有随机值”就忽略缓存一致性要求。

## 上线前做一次故障演练

至少验证以下场景：

1. 删除一个热点 key 后，用压测同时发起请求，观察同一主键的数据库查询次数是否被收敛；
2. 在持锁请求中人为延迟，确认锁过期、token 校验和客户端超时的行为符合预期；
3. 模拟 Redis 不可用，确认应用有明确降级或失败策略，而不是无限重试；
4. 记录锁竞争次数、重建耗时、二次读取命中率和数据库回源量。

互斥锁适合少量高价值热点键。若热点极高、允许短暂旧数据，更实用的方案通常是“逻辑过期 + 后台异步刷新”：读请求继续返回旧值，只让一个后台任务刷新。先根据一致性和可用性目标选择策略，再把它做成可观测、可演练的机制。
