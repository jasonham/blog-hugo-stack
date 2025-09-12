---
title: RESTful API设计规范实用方案
description: 设计 API 时，在公司实际使用中需要遵守的常规规范
date: 2023-08-11T23:43:55+08:00
lastmod: 2023-08-11T23:43:55+08:00
slug: restful-api-design-specification-and-practical-solutions

tags:
  - Django
  - Django REST framework
  - DRF
categories:
  - Python
---

API 是现代 Web 后端开发中非常核心的内容，尤其在面向 C 端用户、高并发、多人协作的系统中，良好的 API 设计会直接影响系统的**可维护性、稳定性、扩展性和前后端协作效率**。

---

## RESTful API 设计规范

REST（Representational State Transfer）是一种软件架构风格，用于设计网络应用程序的接口。RESTful API 是指符合 REST 原则的 API。

### 核心原则：

#### 资源导向（Resource-Oriented）

- 一切皆“资源”，用 URI 表示资源，如：
  ```
  GET  /users           → 获取用户列表
  GET  /users/123       → 获取 ID 为 123 的用户
  POST /users           → 创建新用户
  PUT  /users/123       → 全量更新用户 123
  PATCH /users/123      → 部分更新用户 123
  DELETE /users/123     → 删除用户 123
  ```

#### 使用标准 HTTP 方法表达操作语义

| HTTP 方法 | 语义              | 是否幂等 | 是否安全（无副作用） |
| --------- | ----------------- | -------- | -------------------- |
| GET       | 获取资源          | ✅ 是    | ✅ 是                |
| POST      | 创建资源          | ❌ 否    | ❌ 否                |
| PUT       | 全量更新/替换资源 | ✅ 是    | ❌ 否                |
| PATCH     | 部分更新资源      | ❌ 否\*  | ❌ 否                |
| DELETE    | 删除资源          | ✅ 是    | ❌ 否                |

> \*注：PATCH 是否幂等取决于实现，若每次传相同 patch 内容结果一致，则可视为幂等。

#### 无状态（Stateless）

- 每次请求必须包含所有必要信息，服务端不保存客户端上下文。
- 会话状态应由客户端（如 Token）携带，服务端只负责验证。

#### 使用 HTTP 状态码表达结果状态（见下文“错误码设计”）

#### 资源命名规范

- 使用名词复数：`/users`, `/orders`, `/products`
- 避免动词：❌ `/getUsers`, ✅ `/users`
- 层级清晰：`/users/123/orders` 表示用户 123 的订单

#### 查询参数用于过滤、排序、分页

```
GET /users?status=active&sort=name&limit=10&offset=20
```

#### 返回结构统一

推荐标准响应体结构：

```json
{
  "code": 200,
  "message": "success",
  "data": { ... }  // 或 data: [] 用于列表
}
```

或更简洁地依赖 HTTP 状态码：

```json
{
  "data": { ... },
  "meta": { "pagination": {...} }
}
```

---

## API 版本控制（Versioning）

随着业务迭代，API 会升级，为避免破坏现有客户端（App/Web），必须做版本控制。

### 常见版本控制方式：

#### URI 路径中带版本（最常用）

```
GET /v1/users
GET /v2/users
```

✅ 优点：清晰直观，易调试，CDN/缓存友好  
❌ 缺点：URI 语义被“污染”，不符合“资源唯一标识”原则

#### 请求头中带版本

```
GET /users
Headers: Accept: application/vnd.myapi.v1+json
```

✅ 优点：保持 URI 纯净，符合 REST 理论  
❌ 缺点：调试不便，部分工具/网关支持不友好

#### 参数中带版本（不推荐）

```
GET /users?version=v1
```

❌ 容易被缓存忽略，语义不清晰

#### 域名或子域名区分（大型系统）

```
https://api-v1.example.com/users
https://api-v2.example.com/users
```

✅ 适合大厂做物理隔离部署  
❌ 运维成本高

### 版本演进策略：

- **向后兼容**：v2 应尽量兼容 v1，非破坏性变更（如加字段）可不升级版本
- **废弃策略**：v1 停止维护前应提前通知（如返回 `Deprecated` Header），提供迁移文档
- **默认版本**：未指定版本时默认走最新稳定版 or 最老维护版（根据策略）

---

## 错误码设计（Error Code Design）

良好的错误码体系能帮助前端/客户端快速定位问题，提升用户体验和调试效率。

### 使用标准 HTTP 状态码（第一层语义）

| 状态码  | 含义                              | 说明                      |
| ------- | --------------------------------- | ------------------------- |
| 200     | OK                                | 请求成功                  |
| 201     | Created                           | 资源创建成功              |
| 400     | Bad Request                       | 客户端参数错误            |
| 401     | Unauthorized                      | 未认证（无 Token 或过期） |
| 403     | Forbidden                         | 无权限访问                |
| 404     | Not Found                         | 资源不存在                |
| 409     | Conflict                          | 资源冲突（如重复提交）    |
| 429     | Too Many Requests                 | 请求过于频繁（限流）      |
| 500     | Internal Server Error             | 服务端未知错误            |
| 502/503 | Bad Gateway / Service Unavailable | 网关错误 / 服务暂时不可用 |

### 自定义业务错误码（第二层语义）

在响应体中增加 `code` 字段，用于细化错误类型：

```json
{
  "code": 10001,
  "message": "手机号格式错误",
  "data": null
}
```

或

```json
{
  "error": {
    "code": "USER_PHONE_INVALID",
    "message": "手机号格式错误，请输入11位数字",
    "details": { "field": "phone" }
  }
}
```

### 错误码设计建议：

- **分层设计**：系统级错误码（如网络、权限） + 业务级错误码（如“库存不足”）
- **全局唯一 & 语义化命名**：如 `ORDER_STOCK_NOT_ENOUGH`, `PAYMENT_TIMEOUT`
- **配套文档**：提供错误码对照表，前端可映射为用户友好提示
- **日志关联**：每个错误响应可带 `request_id`，便于后端追踪日志

示例：

```json
{
  "request_id": "req-abc123xyz",
  "code": "PAY_003",
  "message": "支付超时，请重新发起支付",
  "details": {
    "order_id": "ORD202405001",
    "timeout_at": "2024-05-01T10:00:00Z"
  }
}
```

---

## 接口幂等性（Idempotency）

幂等性：**同一个请求，多次执行所产生的影响（副作用）与一次执行相同。**

### 为什么重要？

- 网络抖动、客户端重试、用户重复点击 → 同一请求可能被多次发送
- 若接口不幂等，会导致：重复下单、重复扣款、数据错乱

### 哪些接口必须幂等？

- **支付类接口**
- **创建订单**
- **发放优惠券**
- **修改状态类操作（如“确认收货”）**

### 如何实现幂等？

#### 唯一标识 + 去重表（最常用）

客户端传入唯一幂等键（如 `idempotency_key`），服务端记录已处理的 key。

```http
POST /orders
Headers: Idempotency-Key: order_create_20240501_abc123

Body: { "product_id": 100, "count": 1 }
```

服务端逻辑：

```python
key = request.headers.get('Idempotency-Key')
if redis.exists(key):
    return get_cached_response(key)  # 返回上次结果
else:
    result = create_order(...)
    redis.setex(key, TTL, json.dumps(result))
    return result
```

> 注意：需设置 TTL 避免内存爆炸；需考虑分布式锁避免并发重复处理。

#### 业务字段天然幂等

- 如“更新用户手机号” → 多次执行结果相同
- “将订单状态从‘待支付’改为‘已支付’” → 加状态判断，避免重复更新

#### 数据库唯一约束

- 订单表加唯一索引 `(user_id, product_id, timestamp)` 防重复创建
- 支付流水号全局唯一

#### Token 机制（适用于表单提交）

- 进入页面时生成 token，提交时校验并删除，防止重复提交

---

## 综合示例：一个幂等的创建订单接口

```http
POST /v1/orders
Headers:
  Authorization: Bearer xxx
  Idempotency-Key: order_20240501_usr123_prod456

Body:
{
  "product_id": 456,
  "quantity": 1
}
```

服务端处理：

1. 校验 `Idempotency-Key` 是否已存在 → 存在则直接返回缓存结果
2. 不存在 → 加分布式锁（防并发）
3. 创建订单 → 写入数据库
4. 缓存结果到 Redis（Key = Idempotency-Key, TTL=24h）
5. 释放锁，返回订单信息

响应：

```json
{
  "code": 201,
  "message": "订单创建成功",
  "data": {
    "order_id": "ORD20240501001",
    "status": "pending_payment"
  }
}
```

## 总结

掌握这部分内容，能帮助写出**专业、健壮、易协作**的 API，是资深后端工程师的必备素养。具体实现的方式，可以和公司团队成员协商，根据项目需求和团队风格来选择。一但确定，则列入团队文档，并制定统一标准，让团队成员遵循。
