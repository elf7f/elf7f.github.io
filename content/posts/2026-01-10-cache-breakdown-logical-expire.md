---
title: "缓存击穿怎么解决：逻辑过期方案的理解与实现"
date: 2026-01-10
draft: false
author: ["wenzy"]
tags: ["Redis", "缓存", "缓存击穿", "逻辑过期", "Java", "Spring Boot"]
categories: ["缓存", "问题解决"]
summary: "从热点 Key 场景出发，整理缓存击穿的本质，以及为什么逻辑过期比简单加锁更适合部分高频读业务。"
showToc: true
TocOpen: true
---

## 背景

在做商铺详情、优惠券信息这类高频读取业务时，我一开始的想法很简单：

- 先查 Redis
- Redis 没有就查数据库
- 再把结果写回 Redis

这就是最典型的缓存旁路模式，平时也确实够用。

但只要某个 Key 非常热门，一个问题就会暴露出来：**缓存击穿**。

它不是“缓存完全失效”，也不是“查询不存在的数据”，而是：

> **某个非常热门的 Key 恰好过期了，导致大量请求同时打到数据库。**

如果这个 Key 正好是活动商品、热门店铺、首页推荐这种被频繁访问的数据，数据库压力会在短时间内被迅速放大。

在项目里，我采用的思路是：**对热点数据使用逻辑过期，而不是简单依赖物理 TTL。**

## 先区分三个容易混淆的问题

### 缓存穿透

查询一个本来就不存在的数据。常见处理方式是缓存空值或布隆过滤器。

### 缓存击穿

一个**热点 Key**在某个时刻失效了，导致大量并发请求同时访问数据库。

### 缓存雪崩

大量缓存 key 在同一时间集中失效，导致数据库在短时间内承受成片流量。

我这次重点解决的是第二种：**热点 Key 的缓存击穿**。

## 最直接的方案：互斥锁

处理缓存击穿时，一个经典思路是互斥锁。

流程大概是：

1. 请求发现缓存失效
2. 尝试获取锁
3. 获取到锁的线程去查数据库并重建缓存
4. 其他线程等待或快速失败
5. 缓存重建完成后，后续请求直接读缓存

这个方案的优点很明显：

- 同一时刻只让一个线程回源数据库
- 能有效避免数据库被并发打爆

但它也有代价：

1. 用户请求会等待
2. 热点高峰下容易形成排队
3. 锁的过期、锁误删、锁续期等问题都需要考虑

所以我后来更关注一个更适合“允许短暂旧数据”的方案：**逻辑过期**。

## 什么是逻辑过期

逻辑过期不是让 Redis 真的过期，而是把“过期时间”作为业务字段和数据一起存进去。

例如缓存里存的不只是店铺信息，而是一个包装对象：

```json
{
  "data": {
    "id": 1,
    "name": "Coffee Lab",
    "address": "xxx road"
  },
  "expireTime": "2026-03-06T10:00:00"
}
```

查询时流程变成：

1. 从 Redis 读取缓存对象
2. 判断 `expireTime` 是否已过
3. 如果没过期，直接返回
4. 如果过期了，也先返回旧数据
5. 同时尝试异步重建缓存

这套方案最关键的一点是：

> **即使发现缓存逻辑上过期了，也不立即让大量请求一起回源数据库。**

## 为什么逻辑过期适合热点数据

逻辑过期的核心价值不是“绝对新鲜”，而是“高峰期稳定”。

### 1. 绝大多数请求仍然能快速返回

哪怕缓存已经逻辑过期，只要旧数据还在，当前请求依然可以直接返回结果，不需要等待数据库。

### 2. 只让少量线程去做缓存重建

通常会配合互斥锁或线程池，让一个线程负责重建缓存，其他请求继续读旧值。

### 3. 数据库压力更平滑

不会在过期瞬间突然放大。

所以逻辑过期特别适合这种场景：

- 数据热点很高
- 数据允许短时间旧一点
- 比起绝对实时，更看重服务稳定性

## 一个典型的逻辑过期读取流程

### 第一步：查 Redis

如果 Redis 中没有这个 key，说明缓存根本没建立起来，此时可以走一次普通回源流程。

### 第二步：判断逻辑过期时间

如果未过期，直接返回数据。

### 第三步：如果已过期，尝试获取重建锁

- 获取成功：提交一个异步任务去重建缓存
- 获取失败：说明已有其他线程在重建，当前请求直接返回旧数据

### 第四步：异步线程重建缓存

- 查询数据库
- 重新写入新数据和新的逻辑过期时间
- 释放锁

## 一个简单的数据封装方式

```java
public class RedisData<T> {
    private T data;
    private LocalDateTime expireTime;

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }

    public LocalDateTime getExpireTime() {
        return expireTime;
    }

    public void setExpireTime(LocalDateTime expireTime) {
        this.expireTime = expireTime;
    }
}
```

写缓存时：

```java
public void setWithLogicalExpire(String key, Object value, Long time, TimeUnit unit) {
    RedisData<Object> redisData = new RedisData<>();
    redisData.setData(value);
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
}
```

读取时：

```java
public Shop queryShopWithLogicalExpire(Long id) {
    String key = "cache:shop:" + id;
    String json = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isBlank(json)) {
        return null;
    }

    RedisData<Shop> redisData = JSONUtil.toBean(json, new TypeReference<RedisData<Shop>>() {}, false);
    Shop shop = redisData.getData();
    LocalDateTime expireTime = redisData.getExpireTime();

    if (expireTime.isAfter(LocalDateTime.now())) {
        return shop;
    }

    String lockKey = "lock:shop:" + id;
    boolean isLock = tryLock(lockKey);
    if (isLock) {
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            try {
                rebuildShopCache(id);
            } finally {
                unlock(lockKey);
            }
        });
    }

    return shop;
}
```

这里最重要的不是具体工具类，而是流程上的取舍：**过期了也先返回旧值，再后台更新。**

## 为什么“返回旧数据”是可以接受的

关键在于业务要求。

对于一些强一致、强实时的场景，例如：

- 余额
- 库存最终状态
- 支付结果

显然不能随便返回旧数据。

但对于很多高频只读业务：

- 店铺基础信息
- 商品详情
- 首页展示信息

短时间内旧一点通常是能接受的。

所以逻辑过期不是“更高级的通用方案”，而是**特定场景下更合适的方案**。

## 逻辑过期和互斥锁到底怎么选

### 适合用互斥锁的情况

- 数据必须尽量新
- 可以接受请求等待
- 热点不是特别夸张
- 回源成本可控

### 适合用逻辑过期的情况

- 数据热点很高
- 可以接受短暂旧数据
- 更希望系统稳定、低延迟
- 不希望过期时大量请求排队等待

一个更现实的理解是：

- 互斥锁追求更强一致
- 逻辑过期追求更高可用

## 逻辑过期不是没有锁

逻辑过期方案里通常仍然需要锁。因为如果缓存过期后每个请求都启动一个线程去重建缓存，那一样会造成大量重复回源。

所以常见做法是：

- 用户请求读到逻辑过期数据
- 只有一个线程抢到锁
- 抢到锁的线程去更新缓存
- 其他线程继续返回旧数据

也就是说，**锁仍然存在，只是不再让用户请求阻塞等待锁的结果**。

## 什么时候逻辑过期会不合适

1. 对数据新鲜度要求极高的场景
2. 冷数据
3. 数据体积大且重建成本很高

## 逻辑过期落地时要注意的细节

1. 逻辑过期时间不要全一样，最好有随机偏移。
2. 后台重建应该放入受控线程池。
3. 锁一定要有过期时间，避免异常导致死锁。
4. 空值缓存和逻辑过期是两回事，经常要一起使用。

## 总结

缓存击穿的难点不在于“缓存没了”，而在于：

> **热点数据失效的那个瞬间，会不会把数据库一起拖下水。**

逻辑过期之所以有效，是因为它改变了处理方式：

- 不让所有请求在过期瞬间一起回源
- 允许短暂返回旧数据
- 通过后台异步重建来更新缓存

如果业务可以接受这种取舍，那么它往往比“所有请求一起等锁”更平滑、更稳。

对我来说，这次实践最重要的收获不是记住了一个解决方案，而是更明确地理解了：

**缓存方案从来不是追求单一最优，而是在业务一致性、用户体验和系统稳定性之间找到合适平衡。**
