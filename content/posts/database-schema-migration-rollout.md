---
title: 数据库表结构变更别赌停机窗口：一套可回滚的灰度发布方法
date: 2026-07-27T09:01:20+08:00
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

给线上表加字段、改索引、迁移字段，看起来只是一次 DDL；真正的风险却在应用与数据处于不同版本的那段时间。发布不是原子操作：新旧实例会并存，异步任务可能延后执行，回滚后的旧代码也仍要读取新数据。把“先发 SQL，再发代码”当成固定流程，容易把一次小改动变成不可逆故障。

更稳妥的原则是：**每一步都让旧代码和新代码同时可用，删除动作永远放到最后。** 下面以 Java 服务常见的用户表字段迁移为例，说明一套可执行的展开—迁移—收缩流程。

<!--more-->

## 先识别两类高风险操作

不是所有 DDL 都能直接在高峰期执行。尤其应提前审查：

- 修改列类型、缩短长度、增加 `NOT NULL` 约束；
- 删除列、删除索引、重命名列或表；
- 在大表上创建索引、回填整表数据；
- 一条事务里同时改结构和更新海量历史行。

风险不只来自 SQL 执行时间。长事务或元数据锁会阻塞业务语句；回填过快会挤占数据库 I/O；一旦新代码只认新字段，应用回滚就失去退路。因此，发布前应在接近生产规模的数据上验证执行时间、锁等待和磁盘增长，并准备停止回填的开关。

## 展开：先只做兼容性变更

假设要把 `user_profile.nickname` 迁移为新字段 `display_name`。第一步不要删除旧列，也不要立即给新列加非空限制：

```sql
alter table user_profile
    add column display_name varchar(64) null;

create index idx_user_profile_display_name
    on user_profile (display_name);
```

索引创建的具体锁行为与数据库版本、表引擎和语句选项有关，不能照搬别的环境结论。上线前要用目标数据库版本的测试库执行 `EXPLAIN` 和 DDL 演练；大表应确认是否支持适合在线变更的方式。

应用随后进入“双写、优先读新字段、回退读旧字段”的兼容阶段。写入同一个本地事务，避免新旧值部分成功：

```java
@Transactional
public void updateProfile(long userId, String name) {
    profileRepository.updateNames(userId, name, name);
}

public String displayName(UserProfile profile) {
    return profile.getDisplayName() != null
            ? profile.getDisplayName()
            : profile.getNickname();
}
```

这里的关键不是代码写法，而是发布顺序：先上线能识别两列的代码，再开始填充历史数据。这样即使新列暂时为空，读路径仍可正常工作；若应用需要回滚，旧列也还在。

## 迁移：限速、分批、可重复

历史数据回填不要用一条无界 `update`。按主键范围或批次处理，每批提交一次，并记录进度。任务必须可重复执行：已经填充的行跳过，失败后从检查点继续。

```sql
update user_profile
set display_name = nickname
where id > :lastId
  and id <= :nextId
  and display_name is null;
```

批次大小应由数据库负载决定，而不是写死一个“安全数字”。运行期间至少观察：SQL 延迟、锁等待、主从延迟（如有）、连接池等待和错误率。指标恶化时降低速率或暂停任务；不要让一次后台补数和在线请求争抢全部资源。

双写期间还应做一致性校验，例如统计 `display_name is null and nickname is not null` 的数量，抽样比对两列值，并对双写失败单独报警。没有这些检查，就无法区分“还没迁完”和“代码漏写”。

## 收缩：确认稳定后再移除旧结构

当回填完成、校验通过，并且旧版本应用与延迟任务都已退出后，才进入收缩阶段：先停止写旧列，保留一段观察期；确认没有旧列读取和写入，再删除旧列及相关兼容代码。

```sql
alter table user_profile drop column nickname;
```

删除是不可逆动作，必须有明确的前置条件：备份或可用恢复方案、变更审批、观察指标正常、以及可追溯的版本记录。若业务允许，先只移除应用代码中的读取分支，数据库旧列延后一两个发布周期再删，会比追求“本次发布彻底清理”更安全。

## 把迁移当成一个发布单元

一次结构迁移至少应包含四项交付物：版本化的 DDL 脚本、兼容代码、可暂停的回填任务、以及校验与回滚说明。Flyway、Liquibase 等工具可以帮助记录脚本版本，但不能替你决定发布顺序和数据正确性。

真正可靠的数据库变更，不是 DDL 一次执行成功，而是在新旧应用并存、任务中断、甚至代码回滚时，业务仍能持续提供正确结果。先兼容，再迁移，最后收缩，能把高风险的一次性改表拆成每一步都可验证的工程过程。
