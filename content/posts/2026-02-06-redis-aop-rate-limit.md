---
title: "用 Redis + AOP + 注解实现一个可复用的滑动窗口限流组件"
date: 2026-02-06
draft: false
author: ["wenzy"]
tags: ["Redis", "AOP", "限流", "滑动窗口", "Java", "Spring Boot"]
categories: ["Java后端", "组件设计"]
summary: "从接口保护场景出发，整理为什么选择滑动窗口限流，以及如何结合 Redis、AOP 和自定义注解实现一个通用限流组件。"
showToc: true
TocOpen: true
---

## 背景

在做接口开发时，我一开始对“限流”的理解比较简单，觉得它主要是用来防恶意刷接口。

但真正做项目时会发现，限流的作用远不止防刷。它至少能解决三类问题：

1. 避免单个接口在短时间内被突发请求打垮
2. 控制某类业务的资源消耗
3. 在系统有波动时，优先保护核心服务

尤其是下面这些接口，很适合限流：

- 短信验证码发送
- 登录接口
- 秒杀接口
- 评论、点赞、收藏等频繁操作接口

在项目里，我最后采用的是：**Redis + AOP + 自定义注解**，并且限流算法选择了**滑动窗口**。

## 为什么不直接在业务代码里写限流逻辑

最直接的做法当然是：

- 在 Controller 或 Service 中读取 Redis
- 自己判断当前用户请求次数
- 超过阈值就返回失败

这种写法能用，但有个明显问题：**不够通用**。

如果很多接口都需要限流，你会发现：

- 每个接口都要写类似逻辑
- 业务代码被横切逻辑污染
- 限流规则分散，维护成本高

所以我更倾向于把限流做成一个可复用的组件，让业务层只关心一件事：

> 哪个接口需要限流，用什么维度和规则限流。

## 为什么选择滑动窗口

常见限流算法至少有：

- 固定窗口
- 滑动窗口
- 漏桶
- 令牌桶

我当时选择滑动窗口，主要是因为它在“实现复杂度”和“限流精度”之间比较平衡。

### 固定窗口的问题

例如限制“每分钟 10 次”，用户可以在：

- 00:59 请求 10 次
- 01:00 再请求 10 次

虽然看起来每分钟都没超，但实际上两秒内打了 20 次。

### 滑动窗口的优势

滑动窗口不看自然时间边界，而是看“当前时刻往前推的一段连续时间”。

例如：

- 当前时间往前 60 秒内
- 最多允许 10 次请求

这样统计会更平滑，也更接近真实限流需求。

## 为什么 Redis 适合做限流状态存储

Redis 很适合做这件事，因为：

1. **读写快**
2. **天然支持过期**
3. **可以支持分布式部署**

如果应用有多个实例，限流状态不能只存在单机内存里，否则每个实例都会各算各的。Redis 能很好地做统一状态存储。

## 一个滑动窗口限流的实现思路

在 Redis 里，我比较喜欢用 `ZSet` 实现滑动窗口。

因为 `ZSet` 同时具备：

- 有序性
- 可按 score 范围删除
- 可按区间统计数量

非常适合按时间戳记录请求。

### 基本思路

对于某个限流 key，比如：

```text
rate_limit:user:10086:/api/order/create
```

每来一次请求，就做下面几步：

1. 获取当前时间戳 `now`
2. 删除窗口外的数据
3. 统计窗口内还剩多少请求
4. 如果数量已经超过阈值，则拒绝
5. 否则把当前请求时间戳写入 `ZSet`

## 为什么 key 设计很重要

限流从来不只是“限次数”，更重要的是**限谁**。

我在项目里比较关注三种维度：

### 1. 按 IP 限流

适合未登录接口，例如登录页、验证码发送接口。

### 2. 按用户限流

适合已经登录后的业务接口。

### 3. 按全局限流

适合保护某个非常敏感的核心接口。

同样的限流算法，key 设计不同，表达的业务规则就完全不同。

## 一个典型的 Redis 操作流程

```java
public boolean tryAcquire(String key, long windowSeconds, long maxRequests) {
    long now = System.currentTimeMillis();
    long windowStart = now - windowSeconds * 1000;
    String member = String.valueOf(now);

    stringRedisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);

    Long count = stringRedisTemplate.opsForZSet().zCard(key);
    if (count != null && count >= maxRequests) {
        return false;
    }

    stringRedisTemplate.opsForZSet().add(key, member, now);
    stringRedisTemplate.expire(key, windowSeconds + 5, TimeUnit.SECONDS);

    return true;
}
```

真正上线时最好把这几步进一步收敛为 Lua 脚本执行，避免并发场景下多命令穿插带来的误差。

## 为什么限流逻辑适合放进 AOP

限流本质上是一个典型的横切关注点，和日志、权限、审计很像：

- 不属于核心业务逻辑
- 却需要作用于很多接口
- 希望统一配置和统一处理

所以我更倾向于用自定义注解标记限流规则，再通过 AOP 在方法执行前统一拦截。

## 一个简单的注解设计

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {

    long maxRequests();

    long windowSeconds();

    LimitType limitType() default LimitType.USER;

    String message() default "请求过于频繁，请稍后再试";
}
```

其中 `LimitType` 可以定义为：

```java
public enum LimitType {
    IP,
    USER,
    GLOBAL
}
```

业务层就可以很清晰地声明规则：

```java
@RateLimit(maxRequests = 5, windowSeconds = 60, limitType = LimitType.USER)
@PostMapping("/comment")
public Result publishComment(@RequestBody CommentDTO dto) {
    return Result.ok(commentService.publish(dto));
}
```

## AOP 切面应该做什么

切面通常只做四件事：

1. 读取方法上的限流注解
2. 根据限流维度拼装 key
3. 调用 Redis 限流逻辑
4. 如果超限，直接抛出异常或返回统一失败结果

```java
@Around("@annotation(rateLimit)")
public Object doRateLimit(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
    String key = buildKey(joinPoint, rateLimit);
    boolean allowed = rateLimitService.tryAcquire(
            key,
            rateLimit.windowSeconds(),
            rateLimit.maxRequests()
    );

    if (!allowed) {
        throw new BusinessException(rateLimit.message());
    }

    return joinPoint.proceed();
}
```

## 一个容易忽略的问题：限流失败该怎么返回

限流不仅是技术问题，也涉及用户体验。

我会更倾向于：

- 返回明确提示，而不是模糊的系统错误
- 状态码和业务码区分清楚
- 日志里记录触发限流的 key 和规则

例如：

```json
{
  "code": 42901,
  "message": "请求过于频繁，请 60 秒后再试"
}
```

## 滑动窗口的精度和成本

滑动窗口比固定窗口更准确，但它也不是没有代价。

1. 需要记录更多时间点
2. 高频接口会产生更多 Redis 操作
3. 统计越精细，成本越高

所以滑动窗口很适合做“精度比较重要、规则不算特别极端”的接口保护，但也不是所有场景都必须用它。

## 为什么我觉得“通用性”很重要

真正有价值的，不只是“某个接口能限流”，而是“以后其他接口也能低成本复用”。

所以我会更看重下面几点：

1. 规则通过注解表达
2. 支持多维度 key
3. 算法和业务解耦
4. 统一失败处理

这才是组件化的意义。

## 什么时候不一定要做这么重的限流

不是所有项目都需要 Redis + AOP + 注解这么完整的一套。

如果只是单机小项目，或者只有极少数接口需要简单限频，直接在本地做也能用。

但一旦出现下面这些需求，这套方案就会更有价值：

- 多实例部署
- 限流规则较多
- 接口需要按不同维度限制
- 希望统一治理和复用

## 总结

限流的本质不是“拦住用户”，而是**在资源有限的情况下，让系统优先保证稳定性**。

我之所以采用 Redis + AOP + 注解 + 滑动窗口，是因为它能同时满足几个目标：

- 支持分布式场景
- 规则表达清晰
- 业务侵入小
- 统计比固定窗口更平滑

这次实践让我更明确地理解到：

**好的后端设计，不只是把功能做出来，而是把那些重复出现的系统性问题抽出来，做成可以复用的能力。**
