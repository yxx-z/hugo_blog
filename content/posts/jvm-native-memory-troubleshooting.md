---
title: JVM 堆外内存持续增长：用 NMT 和容器指标定位，而不是只调大堆
date: 2026-08-04T09:00:28+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - JVM
  - 容器
  - 可观测性
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

容器因为 OOMKilled 退出，而 JVM 日志里堆使用率并不高，这通常不是“堆还不够大”，而是进程的总内存超过了容器限制。Java 进程除堆外，还会消耗元空间、线程栈、直接内存、JIT 代码缓存，以及 JNI 或其他 native 库申请的内存。先区分这些来源，再决定改代码、改并发还是改内存限制。

<!--more-->

## 先对齐三个口径

排查前不要把不同口径的数字直接相减。至少同时记录：

- **容器内存**：cgroup 统计的进程实际占用，是 Kubernetes 是否 OOMKilled 的直接依据；
- **进程 RSS**：操作系统驻留物理页，适合观察持续增长趋势；
- **JVM 堆已用量**：应用对象存活量，只覆盖总内存的一部分。

如果堆曲线平稳、RSS 或容器内存持续上升，优先看堆外来源。反过来，RSS 高也不能直接断言“直接内存泄漏”：线程数暴涨带来的栈空间、文件映射和 native 库都可能造成同样现象。

在容器内可以先保留一段连续样本：

```bash
# cgroup v2：查看容器当前内存和限制
cat /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/memory.max

# 进程 RSS、线程数和启动参数；PID 替换为 Java 进程号
ps -o pid,rss,nlwp,args -p <PID>
jcmd <PID> VM.flags
```

`rss` 的单位通常为 KiB，采样时应连同 QPS、线程数和发布版本一起保存。一次快照只能说明“现在很高”，连续趋势才能区分流量峰值和泄漏。

## 用 NMT 给 JVM 内部内存分类

HotSpot 的 Native Memory Tracking（NMT）可以按 JVM 内部分类展示 native 内存，例如 Java Heap、Class、Thread、Code 和 GC。它默认关闭，必须在**进程启动时**打开；线上临时发现问题时，不能靠 `jcmd` 再把它启动起来。

先在一个实例或压测环境增加启动参数：

```bash
JAVA_TOOL_OPTIONS="$JAVA_TOOL_OPTIONS -XX:NativeMemoryTracking=summary"
```

重启后采集基线和增量：

```bash
jcmd <PID> VM.native_memory baseline
# 等待一个可解释的业务周期后执行
jcmd <PID> VM.native_memory summary.diff scale=MB
jcmd <PID> VM.native_memory summary scale=MB
```

`summary.diff` 比一次 `summary` 更有用：它能显示某类内存相对基线的增量。NMT 本身有开销，`summary` 通常适合常态诊断；只有确认问题且需要更细归因时，再在隔离实例启用 `detail`。Oracle 的说明也明确，NMT 跟踪的是 HotSpot/JVM 自身使用的内存，并不能覆盖第三方 native 代码等全部进程内存。因此 NMT 与 RSS 的差额仍然需要结合应用依赖和系统指标解释。

参考：<https://docs.oracle.com/en/java/javase/11/vm/native-memory-tracking.html>

## 按增长类别收敛原因

### Thread 增长：先查线程数和阻塞点

每个 Java 线程通常都要分配本地栈。线程池无界扩张、每个请求新建线程、慢依赖导致工作线程堆积，都会让容器内存先于堆告警。

```bash
jcmd <PID> Thread.print -l > /tmp/thread-dump.txt
# 快速观察线程总数；不要用它替代完整线程转储
ps -o nlwp= -p <PID>
```

修复方向是给线程池设置有界队列和拒绝策略、为远程调用设超时，并把线程名按业务池区分。只调小 `-Xss` 不是通用修复：栈太小可能把问题转化为 `StackOverflowError`，应在压测下验证。

### Class/Metaspace 增长：查动态类加载和 ClassLoader 生命周期

热部署残留、动态代理或脚本引擎若持有旧 ClassLoader，会让类元数据无法卸载。先对比发布前后已加载类数量，并用堆转储检查谁引用了旧 ClassLoader；不要仅通过设置更大的元空间上限掩盖泄漏。

```bash
jcmd <PID> VM.classloader_stats
jcmd <PID> GC.class_histogram > /tmp/class-histogram.txt
```

### 直接内存或 native 差额：回到分配路径

Netty、NIO、压缩库和 JNI 都可能使用堆外内存。对于直接缓冲区，优先确认是否存在未释放的引用、连接堆积或请求体没有限流；对于 JNI/agent/图像处理等库，NMT 未覆盖的 RSS 差额尤其值得检查。此时应将依赖版本、连接数、缓冲区指标与 RSS 增量关联，而不是先把容器 limit 翻倍。

## 一套可执行的处置顺序

1. 先确认 OOM 事件、容器内存曲线和 RSS 是否同步增长；
2. 同时采集线程数、JVM 参数、线程转储和 NMT 差分；
3. 根据增长类别定位到具体线程池、ClassLoader 或 I/O 分配路径；
4. 用压测复现修复前后的增长斜率，而不是只看短时间内“没再 OOM”；
5. 最后才基于峰值、余量和并发模型调整 `-Xmx` 与容器内存限制。

容器 limit 是保护边界，不是内存诊断工具。把堆大小、线程并发和堆外分配放到同一张观测图里，才能避免每次 OOM 都靠扩容临时止血。
