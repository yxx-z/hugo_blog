---
title: Spring Boot 4 升级前，先完成这份可回滚清单
date: 2026-07-18T13:31:15+08:00
draft: false
author:
  name: yxx
  link:
  email:
  avatar:
description: 从 Spring Boot 3.x 升级到 4.x 前，如何通过依赖、配置、测试和发布节奏降低风险。
keywords:
  - Spring Boot 4
  - 升级
  - Java
  - 工程实践
license:
comment: false
weight: 0
tags:
  - Spring Boot
  - 工程实践
  - 升级
categories:
  - java
hiddenFromHomePage: false
hiddenFromSearch: false
summary: 从 3.x 升级到 Spring Boot 4，不应只改版本号。本文给出一套以可回滚和可验证为核心的升级清单。
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url:
---

Spring Boot 大版本升级最常见的误区，是把它理解成一次 Maven 版本替换：改掉 `spring-boot.version`，能编译就合并。

这不够。真正的风险往往出现在运行期：自动配置变化、依赖管理升级、废弃 API 删除、测试替身失效，以及配置项悄悄不再生效。一次可控升级应该以“能回滚、能比较、能定位”为目标。

<!--more-->

## 先明确升级目标

升级前先写清楚三个问题：

1. **为什么现在升级？** 是安全修复、依赖兼容、性能需求，还是希望使用新能力？
2. **升级范围是什么？** 只升级 Spring Boot，还是同时升级 JDK、Spring Cloud、数据库驱动和部署镜像？
3. **失败后怎么退？** 回退到哪个 Git 提交、哪个制品、哪套配置？

如果第三个问题没有答案，不要直接在生产分支上做升级。

Spring 官方迁移指南建议，进入 Boot 4 前先升到最新的 3.5.x，并处理 3.x 中的废弃项。这个顺序很重要：把“旧版本遗留问题”和“新主版本变更”分开，排查成本会低很多。

## 第一阶段：在 3.5.x 消灭废弃项

先保持 JDK 和业务依赖不动，只把应用升级到当前可用的 3.5.x。随后完成：

- 编译时开启并处理 deprecation warning；
- 搜索项目内的 `@Deprecated` 调用；
- 对照依赖树，确认没有被间接拉入过旧的 Spring 组件；
- 运行单元测试、集成测试和关键接口冒烟测试；
- 记录当前启动耗时、堆内存、错误率和核心接口 P95，作为升级后的对比基线。

Maven 项目至少应保存两份依赖树：升级前和升级后。

```bash
mvn -q dependency:tree -DoutputFile=target/dependency-tree.txt
mvn -q help:effective-pom -Doutput=target/effective-pom.xml
```

很多“代码没改却启动失败”的问题，根因不在 Spring Boot 本身，而是 BOM 带动了 Jackson、Hibernate、Tomcat、Netty、测试框架或数据库驱动的版本变化。

## 第二阶段：用独立分支做 Boot 4 试迁移

不要在长期开发分支里直接改。建立独立升级分支，例如：

```bash
git switch master
git pull --ff-only origin master
git switch -c feature/upgrade-spring-boot-4
```

升级改动要尽可能小：先改 Spring Boot 父版本或 BOM，再按构建错误逐项处理。不要趁机混入业务重构、表结构调整和接口改版，否则失败时无法判断是哪类改动造成的。

建议的处理顺序：

1. **编译失败**：优先处理包名、移除 API、方法签名变化；
2. **测试失败**：区分测试代码过期和真实行为变化；
3. **启动失败**：检查自动配置报告、条件 Bean、配置绑定和依赖冲突；
4. **运行行为变化**：再检查序列化、数据库方言、事务、鉴权、观测和网络超时。

## 第三阶段：重点检查四类高风险点

### 1. 配置项不再生效

配置最危险，因为应用可能正常启动，但使用了默认值。

升级期应临时加入 `spring-boot-properties-migrator`，让启动日志报告失效或改名的配置项。它是迁移辅助工具，不建议长期保留在生产依赖中。

同时检查：

- 环境变量是否仍符合 relaxed binding；
- 自定义 `@ConfigurationProperties` 是否完整绑定；
- profile 覆盖顺序是否改变；
- 安全、CORS、连接池、日志级别等关键配置是否实际生效。

### 2. 测试替身与测试上下文

Spring Boot 4 移除了部分旧的测试支持。升级后不要为了让测试变绿而盲目扩大 Mock 范围。

优先保留两层测试：

- **纯单元测试**：不启动 Spring，验证业务规则；
- **关键集成测试**：启动最小上下文，验证配置、数据库访问、鉴权和核心接口链路。

对于外部 HTTP、消息队列、邮件等依赖，用 Testcontainers、MockWebServer 或明确的 fake 实现隔离，避免测试结果取决于某台共享环境机器。

### 3. 依赖管理与重复版本

不要在子模块随意覆盖 Spring Boot 管理的基础依赖版本。升级后执行：

```bash
mvn dependency:tree -Dverbose
mvn enforcer:enforce
```

重点看是否同时存在多个版本的 Jackson、SLF4J、Servlet API、Hibernate 或 Spring Framework。能由 Boot BOM 管理的，尽量交还给 BOM。

### 4. 生产发布与回滚

升级成功不是本地 `mvn test` 成功，而是灰度环境的真实流量表现正常。

一个最小发布闭环是：

```text
升级分支
  -> CI 完整构建与测试
  -> 测试环境验证
  -> 小流量/单实例灰度
  -> 观察指标
  -> 扩容发布
```

回滚必须使用上一个已经验证过的制品，而不是临时从分支重新构建。数据库迁移如果不可逆，要和应用升级拆开发布，或设计向后兼容窗口。

## 可直接复用的验收清单

上线前逐项确认：

- [ ] 已升级并稳定运行在最新 3.5.x；
- [ ] 已处理 3.x 的废弃 API 与配置警告；
- [ ] 已比较升级前后的依赖树；
- [ ] 单元测试、集成测试、接口冒烟测试均通过；
- [ ] 已验证认证、事务、序列化、数据库连接和定时任务；
- [ ] 已验证日志、指标、TraceId 和告警；
- [ ] 有可直接部署的上一稳定版本制品；
- [ ] 灰度监控指标与升级前基线可比较；
- [ ] 数据库变更具备回退或兼容方案。

## 结语

升级 Spring Boot 4 的核心不是追最新版本，而是建立一次可复制的升级能力。把依赖比较、配置校验、分层测试、灰度观察和回滚制品固化到流水线里，下一次升级就不再是“赌一把能不能启动”。

参考：

- [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes)
