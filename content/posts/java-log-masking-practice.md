---
title: Java 日志脱敏别只靠正则：把敏感数据挡在记录边界之外
date: 2026-08-05T09:00:49+08:00
draft: false
author:
  name: yxx
tags:
  - Java
  - Spring Boot
  - 安全
  - 工程实践
categories:
  - java
toc: true
math: false
lightgallery: false
comment: false
---

日志能帮助定位问题，也很容易变成数据泄露的扩散面。一次看似无害的 `log.info("request={}", body)`，可能把手机号、身份证号、访问令牌或银行卡号复制到应用日志、日志平台、备份和排障导出文件中。问题发生后，再去清理历史副本通常代价很高。

可靠的原则不是“发现敏感字段后再打码”，而是先定义哪些数据不允许进入日志，再在请求入口、业务日志和异常链路设置边界。脱敏是纵深防御的一层，不能替代权限控制和最小化采集。

<!--more-->

## 先列出禁止记录的数据

不同系统的清单不同，但至少应把以下内容列入禁止原文记录的范围：

- 密码、短信验证码、支付口令和私钥；
- `Authorization`、Cookie、会话标识、刷新令牌和第三方 API 密钥；
- 身份证号、银行卡号、完整手机号和精确住址；
- 用户提交的原始文件内容，以及包含上述字段的请求体。

这里的关键是区分“业务需要保存”和“排障需要打印”。例如订单服务可以保存收货地址，但大多数错误日志只需要订单号、字段名和校验失败原因。日志字段应当为诊断服务，而不是充当请求数据备份。

## 在入口移除高风险请求信息

Spring Boot 项目常用过滤器记录方法、路径和耗时。过滤器中不要直接记录全部请求头，也不要为了排查方便读取并打印请求体。下面的示例只记录经过筛选的请求头，并给每个请求生成关联 ID：

```java
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
public class RequestLogFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        String requestId = request.getHeader("X-Request-Id");
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString();
        }

        long startedAt = System.nanoTime();
        MDC.put("requestId", requestId);
        try {
            filterChain.doFilter(request, response);
        } finally {
            long elapsedMs = (System.nanoTime() - startedAt) / 1_000_000;
            logger.info("method={} path={} status={} elapsedMs={} userAgent={}",
                    request.getMethod(), request.getRequestURI(), response.getStatus(),
                    elapsedMs, safeUserAgent(request.getHeader("User-Agent")));
            MDC.remove("requestId");
        }
    }

    private String safeUserAgent(String value) {
        if (value == null) {
            return "-";
        }
        return value.length() <= 120 ? value : value.substring(0, 120);
    }
}
```

`X-Request-Id` 是外部可控输入，不能把它当作可信身份字段。若接收上游传来的 ID，应限制长度和字符集；追踪用户身份时，记录内部用户 ID 或不可逆的业务标识，不记录令牌本身。

## 业务日志使用白名单字段

最容易失控的是对象直接序列化。`log.debug("login request={}", request)` 会随着 DTO 字段增加，悄悄把新字段写入日志。更稳妥的方式是明确列出允许出现的字段：

```java
log.info("createOrder requestId={} orderId={} itemCount={} channel={}",
        MDC.get("requestId"), orderId, command.itemCount(), command.channel());
```

对必须展示给客服或运维的手机号，可以在展示前处理，而不是让原值流入格式化器：

```java
static String maskPhone(String phone) {
    if (phone == null || phone.length() < 7) {
        return "***";
    }
    return phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
}
```

该方法只适合展示，不是加密。脱敏后的值仍可能结合其他数据识别个人，因此日志查询权限、导出权限和保留期限仍需按数据分级执行。

## 异常日志也要控制上下文

`logger.error("call failed", exception)` 应保留，因为堆栈对定位故障很重要；风险通常来自异常消息拼接的上下文。不要写 `logger.error("payment failed, token={}", token, exception)`。记录稳定的资源标识、下游名称、HTTP 状态和错误码即可。

全局异常处理器返回客户端的消息也应与内部异常分开。客户端只获得可理解的错误码和请求 ID，详细堆栈留在受控日志中。这样既避免把 SQL、内部地址或参数回显给调用方，也让排障人员能用请求 ID 关联日志。

## 用规则扫描做最后一道防线

日志采集端可以增加正则掩码或告警，例如识别 `Bearer `、邮箱和疑似银行卡号。这一层适合拦截遗漏，不能作为唯一方案：正则会漏掉新字段，也可能误伤正常内容。生产变更前，建议把以下检查放进测试或发布验收：

1. 为登录、注册、支付和文件上传接口构造带密码、令牌、手机号的请求；
2. 搜索应用控制台和采集后的日志，确认原值不存在；
3. 检查异常分支与 debug 日志，而非只检查成功路径；
4. 验证日志平台中只有需要排障的角色可以查询和导出。

脱敏规则需要跟着字段和接口演进。每新增一个包含个人信息或凭据的 DTO，就应同时决定：它是否需要记录、允许记录哪些派生字段、以及对应的自动化验证如何覆盖。这比故障发生后在海量日志里补救更可控。
