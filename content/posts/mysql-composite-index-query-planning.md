---
title: MySQL 联合索引不生效？先从查询条件与执行计划定位
date: 2026-08-03T09:01:01+08:00
draft: false
author:
  name: yxx
tags:
  - MySQL
  - 数据库
  - SQL
  - 性能优化
categories:
  - database
toc: true
math: false
lightgallery: false
comment: false
---

给订单列表接口加了联合索引，SQL 却还是慢，常见处理是继续加索引。这往往会让写入更重、索引更多，却没有解决真正的问题。联合索引是否能缩小扫描范围，取决于查询条件、排序方式和数据分布，而不是列是否“都在索引里”。

下面以一个订单表为例，建立一套先验证、再修改的排查过程。

<!--more-->

## 先固定问题 SQL

假设运营后台按租户、状态和创建时间查询订单：

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  tenant_id BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at DATETIME NOT NULL,
  amount DECIMAL(12, 2) NOT NULL,
  KEY idx_tenant_status_created (tenant_id, status, created_at)
) ENGINE=InnoDB;

SELECT id, created_at, amount
FROM orders
WHERE tenant_id = 42
  AND status = 'PAID'
  AND created_at >= '2026-08-01 00:00:00'
ORDER BY created_at DESC
LIMIT 50;
```

这个索引的列顺序并非背口诀，而是服务这条访问路径：前两个等值条件定位到较小范围，随后按 `created_at` 做范围扫描；索引中的时间顺序也可以满足排序，因此通常不必额外排序。`id` 对 InnoDB 二级索引是可用的主键值，但 `amount` 不在该索引中，读取它仍可能需要回表。不要在没有测量前，为“覆盖索引”盲目塞入很多列。

## 用 EXPLAIN ANALYZE 看真实执行

先在与生产数据分布接近的环境执行：

```sql
EXPLAIN ANALYZE
SELECT id, created_at, amount
FROM orders
WHERE tenant_id = 42
  AND status = 'PAID'
  AND created_at >= '2026-08-01 00:00:00'
ORDER BY created_at DESC
LIMIT 50;
```

重点不是只看是否出现索引名，而是核对：

- 实际扫描行数是否远大于最终返回的 50 行；
- 是否出现额外排序步骤；
- 耗时主要在索引扫描、回表，还是排序；
- 预估行数和实际行数是否差距很大。

`EXPLAIN ANALYZE` 会实际执行语句，不能对未知成本的写操作直接使用；排查线上慢查询时，应先从监控或慢日志拿到参数化后的真实条件，再在安全环境复现。

## 联合索引为何会“看起来没用”

第一个典型问题是跳过最左列：

```sql
-- 无法直接按 idx_tenant_status_created 的 status 前缀定位
WHERE status = 'PAID' AND created_at >= '2026-08-01 00:00:00'
```

第二个问题是范围条件截断后续列的有序利用。若索引为 `(tenant_id, created_at, status)`，查询中的 `created_at >= ...` 已经是范围，`status` 通常不能再像连续等值前缀那样帮助进一步缩小索引区间。因此对上面的固定查询，`(tenant_id, status, created_at)` 更贴合。

第三个问题是对索引列施加函数或隐式类型转换：

```sql
-- 不推荐：需要计算每行日期，且无法直接按 created_at 的范围定位
WHERE DATE(created_at) = '2026-08-01'

-- 改为半开区间
WHERE created_at >= '2026-08-01 00:00:00'
  AND created_at <  '2026-08-02 00:00:00'
```

字符串列和数值参数类型不一致也可能改变比较方式。应用侧应绑定正确类型，而不是依赖数据库临时转换。

## 索引不是脱离业务的固定答案

索引顺序必须匹配高频查询，而不是表中字段的“重要程度”。如果绝大多数请求只按 `tenant_id` 和时间筛选，状态选择性很低，那么 `(tenant_id, created_at)` 可能更合适；反过来，如果状态能显著过滤数据，保留它在时间之前才有意义。结论要由实际行数、延迟和写入成本共同决定。

修改前先统计现有索引与查询样本，避免创建功能重复的索引。上线后持续观察慢查询数量、P95/P99 延迟、写入延迟和索引体积；确认新索引稳定收益后，再通过正式变更流程评估是否移除冗余索引。

## 一份可执行的排查清单

1. 从慢日志或监控获取真实 SQL、参数与频率；
2. 用 `EXPLAIN ANALYZE` 对照实际扫描行数和耗时；
3. 让等值过滤列、范围列、排序列按查询路径排列；
4. 改写函数条件和类型不一致，而不是先强制索引；
5. 在接近生产的数据集验证，再灰度观察读写两侧指标。

联合索引优化的核心不是“让优化器一定选某个索引”，而是让索引结构与请求路径一致，并用执行计划证明扫描范围确实变小。
