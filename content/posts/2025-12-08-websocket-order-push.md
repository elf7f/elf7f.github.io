---
title: "用 WebSocket 实现订单状态实时推送"
date: 2025-12-08
draft: false
author: ["wenzy"]
tags: ["WebSocket", "Java", "Spring Boot", "实时推送", "订单系统"]
categories: ["Java后端", "实时通信"]
summary: "从订单状态更新场景出发，整理为什么轮询不够好，以及如何用 WebSocket 改善用户侧的实时感知和交互体验。"
showToc: true
TocOpen: true
---

## 背景

在订单类业务里，用户往往不只关心“下单成功没成功”，还关心后续状态是不是发生了变化。

比如：

- 是否已接单
- 是否正在制作
- 是否配送中
- 是否已完成

如果用户每次都必须手动刷新页面才能看到最新状态，体验通常会比较差。尤其在订单流程比较短、状态变化比较频繁的场景里，这种问题会更明显。

所以在项目里，我尝试使用 **WebSocket** 来做订单状态实时推送，让前端可以在状态变化时主动收到通知，而不是不停轮询接口。

## 为什么轮询不是最理想的方案

最常见的办法当然是轮询：

- 前端每隔几秒请求一次订单详情接口
- 发现状态变了就更新页面

这种方式实现简单，也很容易上手，但它有几个明显缺点。

1. **有延迟**：如果 5 秒轮询一次，那么状态变化最多可能要等 5 秒用户才能看到。
2. **有浪费**：很多轮询请求拿到的结果其实根本没变，本质上是在做无效请求。
3. **用户多时压力会放大**：如果一个页面上有很多在线用户，每个人都在固定频率轮询，那么服务器会承受很多并不真正必要的查询流量。

所以轮询虽然“能用”，但并不够优雅。

## WebSocket 为什么更适合这种场景

WebSocket 最核心的价值在于：

> 它建立了一条客户端和服务端之间的长连接，服务端可以在需要的时候主动把消息推给客户端。

对于订单状态推送来说，这种模式非常自然：

- 用户打开订单页面时建立连接
- 后端订单状态一旦变化
- 服务端主动推送一条消息
- 前端立刻更新页面

这样做的好处很明显：

1. **实时性更强**
2. **请求更少**
3. **用户体验更好**

## 一个典型的应用场景

以外卖订单为例，状态可能按下面的路径变化：

```text
待支付 → 已支付 → 已接单 → 配送中 → 已完成
```

如果前端只靠轮询，用户可能会觉得状态变化不够及时。而如果在每次状态变更后都由服务端主动推送，页面体验会更顺滑。

## WebSocket 的基本流程

### 第一步：客户端建立连接

前端打开订单页面时，创建 WebSocket 连接，例如：

```javascript
const socket = new WebSocket("ws://localhost:8080/ws/order");
```

### 第二步：服务端保存连接会话

后端在连接建立成功后，保存当前用户和 WebSocket 会话的映射关系。

### 第三步：订单状态变化时主动推送消息

当订单服务把订单状态从 A 改成 B 时，触发推送逻辑。

### 第四步：前端接收消息并更新页面

收到消息后解析 JSON，更新当前订单状态。

## 服务端最关键的问题：怎么知道消息该推给谁

订单状态推送不是广播消息，而是**精准推送给对应用户**。

所以服务端至少要解决一个问题：

> 当前这个 WebSocket 会话属于哪个用户？

常见做法是：

- 建立连接时携带 Token 或用户 id
- 服务端在握手或连接初始化阶段解析身份
- 建立 `userId -> Session` 的映射关系

例如可以维护一个线程安全 Map：

```java
private static final ConcurrentHashMap<Long, Session> SESSION_MAP = new ConcurrentHashMap<>();
```

连接建立时放进去，连接断开时移除。

这样订单状态变化时，只要知道这个订单属于哪个用户，就能找到对应连接进行推送。

## 一个简单的后端示例思路

```java
@ServerEndpoint("/ws/order/{userId}")
@Component
public class OrderWebSocketServer {

    private static final ConcurrentHashMap<Long, Session> SESSION_MAP = new ConcurrentHashMap<>();

    @OnOpen
    public void onOpen(Session session, @PathParam("userId") Long userId) {
        SESSION_MAP.put(userId, session);
    }

    @OnClose
    public void onClose(@PathParam("userId") Long userId) {
        SESSION_MAP.remove(userId);
    }

    public static void sendMessage(Long userId, String message) {
        Session session = SESSION_MAP.get(userId);
        if (session != null && session.isOpen()) {
            session.getAsyncRemote().sendText(message);
        }
    }
}
```

当订单状态变化时，可以调用：

```java
OrderWebSocketServer.sendMessage(userId, "{"orderId":1001,"status":"DELIVERING"}");
```

这里的代码不复杂，真正重要的是**连接管理和消息路由**这两个思路。

## 推送消息的数据格式应该怎么设计

我更倾向于用结构化 JSON。

```json
{
  "type": "ORDER_STATUS_CHANGED",
  "orderId": 1001,
  "status": "DELIVERING",
  "message": "订单正在配送中"
}
```

这样前端可以根据 `type` 判断如何处理，也方便以后扩展更多实时事件。

## 前端接收到消息后怎么处理

```javascript
socket.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.type === "ORDER_STATUS_CHANGED") {
    updateOrderStatus(msg.orderId, msg.status);
  }
};
```

也就是说，WebSocket 的价值主要在“把消息送到”，而不是替代所有前端状态管理。

## 实时推送不代表不需要接口查询

很多人会把 WebSocket 理解成“用了它就不需要 HTTP 了”，其实不是。

在订单系统里，两者通常是配合关系：

- **HTTP 接口**：负责初始化页面、获取订单详情、拉取历史数据
- **WebSocket**：负责推送增量变化

## 这类方案的几个现实问题

1. **连接管理**：在线用户多了之后，服务端需要维护大量长连接。
2. **断线重连**：网络波动、浏览器切后台、移动端切网，都会导致连接断开。
3. **多实例部署**：用户连接在 A 实例上，而订单状态更新事件发生在 B 实例上，需要额外的消息分发机制。
4. **鉴权**：不能只靠前端传一个 userId 就相信它是谁，建立连接时最好结合 Token 做身份校验。

## 为什么订单状态推送特别适合 WebSocket

1. **状态变化是事件驱动的**
2. **用户对实时感有明显感知**
3. **推送内容小而频率适中**

所以相比把 WebSocket 用在一些不太必要的地方，订单状态推送是一个比较自然、也比较容易体现价值的场景。

## 我对实时推送的理解变化

在接触 WebSocket 之前，我会把“前端更新状态”理解成一个纯前端问题：页面想办法拿到最新数据就行。

但实际做下来之后，我觉得它更像是一个完整链路问题：

- 后端什么时候认定状态变化
- 如何把变化转换成事件
- 如何找到对应用户连接
- 如何把消息稳定送达
- 前端如何根据事件更新页面

也就是说，实时推送不是一个 API 替换，而是一次通信模式改变。

## 总结

用 WebSocket 做订单状态推送，本质上是在解决一个问题：

> **让用户在真正发生变化时，第一时间感知到，而不是靠不断轮询去“猜”。**

和轮询相比，它的优势主要在于：

- 实时性更好
- 无效请求更少
- 用户体验更自然

当然，它也带来了连接管理、鉴权、多实例分发这些新的问题。但如果业务场景本身就是典型的事件驱动型更新，那么 WebSocket 依然是非常值得使用的方案。

这次实践让我更明确地理解到：

**很多时候，提升用户体验并不是多写一个接口，而是换一种更适合业务本质的通信方式。**
