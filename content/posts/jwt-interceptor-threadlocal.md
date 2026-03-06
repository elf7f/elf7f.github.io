---
title: "JWT + 拦截器 + ThreadLocal：一次登录校验链路整理"
date: 2025-11-02
draft: false
author: ["wenzy"]
tags: ["Java", "JWT", "Spring MVC", "拦截器", "ThreadLocal", "认证"]
categories: ["Java后端", "认证授权"]
summary: "围绕一个前后端分离项目，整理 JWT 无状态认证为什么适合接口场景，以及拦截器和 ThreadLocal 如何配合完成用户身份校验。"
showToc: true
TocOpen: true
---

## 背景

在做前后端分离项目时，用户登录之后，后端必须解决一个核心问题：

> 后续每一个请求，系统怎么知道“你是谁”？

最传统的办法是 Session。用户登录后，服务端保存登录状态，客户端带着 Cookie，后端再去 Session 中查用户信息。

但在前后端分离、接口化开发的场景里，我后来更倾向于采用：

- **JWT**：负责保存用户身份凭证
- **Spring MVC 拦截器**：负责在请求进入业务前校验登录态
- **ThreadLocal**：负责在当前请求生命周期内传递用户信息

## 为什么 Session 不是唯一答案

Session 当然能用，而且在很多传统 Web 项目里仍然很合适。但前后端分离后，它会有几个明显特点：

1. **服务端要保存状态**
2. **接口调用更依赖 Cookie**
3. **跨服务或网关扩展时不够轻量**

所以在我的项目里，选择 JWT 更符合接口化开发场景。

## JWT 到底是什么

JWT 的核心思想其实不复杂：

> 把用户身份信息和一些声明打包成一个签名后的 Token，客户端后续请求时携带它，服务端验证签名后就能确认身份。

它一般由三部分组成：

```text
Header.Payload.Signature
```

例如：

- Header：声明算法类型
- Payload：存放用户 id、过期时间等信息
- Signature：用密钥对前两部分进行签名

## 为什么 JWT 适合前后端分离

1. **无状态**
2. **接口传递方便**
3. **更适合多端接入**

当然，JWT 也不是没有代价，比如撤销、续期、过期处理都要自己设计。但对于学习实践项目和一般接口认证链路，它是很常见的方案。

## 一条登录校验链路是怎么跑起来的

### 第一步：用户登录

用户提交账号信息，服务端校验成功后，生成一个 JWT。

### 第二步：前端保存 Token

前端把 Token 放在本地存储或内存中，后续请求统一带上。

### 第三步：后端拦截请求

请求进入系统后，先经过拦截器：

1. 读取请求头中的 Token
2. 校验 Token 是否存在、是否合法、是否过期
3. 从 Token 中解析用户信息
4. 把用户信息放到当前线程上下文
5. 放行到 Controller / Service

### 第四步：业务代码读取当前用户

业务层不需要每个方法都自己解析 Token，只需要从 ThreadLocal 中获取当前用户即可。

## 为什么登录校验适合放在拦截器里

如果不使用拦截器，最直接的写法就是每个接口都手动做这些事：

- 读请求头
- 解析 Token
- 判断是否登录
- 获取用户 id

很快你就会发现：

- 到处都是重复代码
- 业务逻辑和认证逻辑混在一起
- 后期维护成本很高

而拦截器非常适合做这种“在请求进入 Controller 之前统一处理”的事情。

## 一个典型的拦截器逻辑

```java
public class JwtTokenInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null || token.isBlank()) {
            response.setStatus(401);
            return false;
        }

        Claims claims;
        try {
            claims = JwtUtil.parseToken(token);
        } catch (Exception e) {
            response.setStatus(401);
            return false;
        }

        Long userId = Long.valueOf(claims.get("userId").toString());
        UserContext.set(userId);

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) throws Exception {
        UserContext.remove();
    }
}
```

最重要的是两件事：

- `preHandle` 中解析并保存用户信息
- `afterCompletion` 中清理 ThreadLocal

## ThreadLocal 在这里到底做了什么

ThreadLocal 在这里的用途其实非常朴素：

> 在同一个请求处理线程里，保存一份当前用户信息，方便下游代码随时读取。

```java
public class UserContext {

    private static final ThreadLocal<Long> USER_HOLDER = new ThreadLocal<>();

    public static void set(Long userId) {
        USER_HOLDER.set(userId);
    }

    public static Long get() {
        return USER_HOLDER.get();
    }

    public static void remove() {
        USER_HOLDER.remove();
    }
}
```

这样 Controller 或 Service 层只要：

```java
Long userId = UserContext.get();
```

就能拿到当前登录用户，不需要层层传参。

## 为什么不用参数一直往下传

当然也可以不用 ThreadLocal，而是在 Controller 里拿到用户 id 后，一层层传给 Service、再传给下游方法。

但这种写法会有两个问题：

1. 与业务逻辑耦合过深
2. 代码会变得很啰嗦

所以在请求级上下文场景里，ThreadLocal 是一个很自然的选择。

## ThreadLocal 最大的注意点：一定要清理

ThreadLocal 好用，但也有一个绝对不能忽略的问题：

> 用完一定要 remove。

因为 Web 容器通常会复用线程池中的线程。如果一个请求结束后没有清理当前线程中的用户信息，下一个请求复用这个线程时，就可能读到上一个用户的数据。

所以在用 ThreadLocal 时，最强调的一点就是：

- 设置时在拦截器 `preHandle`
- 清理时在 `afterCompletion`

## JWT 方案的常见问题

### 1. Token 过期怎么处理

JWT 通常会设置有效期。过期后：

- 要求用户重新登录
- 或者通过刷新机制重新获取新 Token

### 2. Token 被盗怎么办

所以必须配合：

- HTTPS
- 合理的有效期
- 敏感接口校验
- 前端安全存储策略

### 3. 用户主动退出登录怎么办

纯 JWT 是无状态的，这意味着服务端默认不保存登录状态。用户退出时，如果要立即失效某个 Token，就需要额外做黑名单或版本号控制。

## 为什么很多项目会“JWT + Redis”一起用

因为纯 JWT 虽然无状态，但也带来一个问题：服务端不太方便主动控制 Token 失效。

所以很多项目会选择：

- JWT 负责身份声明
- Redis 负责保存登录态版本、黑名单或刷新信息

这样既有 JWT 的轻量性，也保留了一定的服务端控制能力。

## 什么时候这套方案很合适

它特别适合下面这些情况：

- 前后端分离项目
- RESTful API 风格开发
- 用户身份信息主要用于接口认证
- 希望登录校验逻辑统一、清晰

## 这套方案给我的最大收获

我以前会把“登录功能”理解成：

- 输账号密码
- 验证成功
- 返回一个 Token

后来真正整理整个链路之后，才更清楚地看到它其实包含三个层次：

1. **身份声明**：由 JWT 负责
2. **统一校验**：由拦截器负责
3. **请求上下文传递**：由 ThreadLocal 负责

也正因为职责拆得清楚，整个链路才会既简洁又好维护。

## 总结

JWT + 拦截器 + ThreadLocal 这套组合，在前后端分离项目里非常实用。

它解决的问题并不只是“登录成功后怎么继续访问”，而是：

- 如何让身份信息无状态地随请求传递
- 如何把登录校验从业务代码里抽出来
- 如何在当前请求上下文中方便地拿到用户信息

对我来说，这次实践最大的理解变化是：

**认证不是一个单点功能，而是一条完整的请求处理链路。只有把这条链路拆开看清楚，很多设计选择才会变得自然。**
