---
title: 常用授权机制
description: 主要介绍了JWT/OAuth2认证授权机制、防刷、防重放、接口安全加固
date: 2023-11-13T00:01:58+08:00
lastmod: 2023-11-13T00:01:58+08:00
slug: common-authorization-mechanisms

tags:
  - Python
categories:
  - Python
---

鉴权是现代 Web 系统（尤其是面向 C 端用户的高并发系统）中**安全架构的核心内容**。  
公司内部前后端分离的 Web 系统，一般会采用 JWT 认证机制。  
涉及第三方的系统，一般会采用 OAuth2 认证机制。  
考虑到安全的因素，在设计架构时，需要考虑到防刷、防重放、注意接口安全加固。

## JWT（JSON Web Token）认证机制

### 什么是 JWT？

JWT 是一种开放标准（RFC 7519），用于在各方之间**安全地传输信息**作为 JSON 对象。常用于**无状态身份认证**。

一个 JWT 由三部分组成，用 `.` 分隔：

```
Header.Payload.Signature
```

- **Header**：算法（如 HS256、RS256）和类型（JWT）
- **Payload**：用户信息（如 user_id、role、exp 过期时间等）
- **Signature**：用密钥对前两部分签名，防止篡改

示例：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### JWT 工作流程

1. 用户登录 → 服务端验证账号密码 → 生成 JWT 返回给客户端
2. 客户端后续请求在 `Authorization: Bearer <token>` 中携带 JWT
3. 服务端验证签名、检查有效期、提取用户信息 → 授权访问

### 优点

- 无状态 → 适合分布式/微服务架构
- 自包含 → 减少数据库查询
- 跨域友好 → 适合前后端分离、App、小程序等

### 缺点 & 风险

- 无法主动失效（除非引入黑名单或短过期时间）
- Payload 未加密（只是 Base64 编码，可被解码）→ 敏感信息不能放里面
- 若签名密钥泄露 → 所有 Token 可伪造

### 安全加固建议

使用 HTTPS 防止中间人窃取  
 设置合理的过期时间（access_token 短 + refresh_token 长）  
 不在 Payload 中放敏感信息（如密码、身份证号）  
 使用强签名算法（如 RS256 比 HS256 更安全）  
 实现 Token 黑名单（用于强制登出或封禁）  
 校验 `iss`（签发者）、`aud`（受众）防止 Token 被其他服务误用

---

## OAuth2.0 授权机制

### 什么是 OAuth2？

OAuth2 是一种**授权框架**（不是认证协议），允许第三方应用在用户授权后访问其资源，而无需获取用户密码。

常见场景：

- “用微信登录”
- “允许某 App 访问你的 Google 日历”

### 四种授权模式（常用两种）

#### 授权码模式（Authorization Code）——最安全，用于 Web/App

1. 用户跳转到授权服务器（如微信）
2. 用户同意授权 → 返回授权码 `code` 给客户端
3. 客户端用 `code` + `client_secret` 向授权服务器换 `access_token`
4. 用 `access_token` 访问用户资源

适合有后端的 Web 应用，避免 Token 暴露在前端

#### 密码模式（Resource Owner Password Credentials）——慎用

用户直接提供账号密码给客户端 → 客户端换取 Token  
⚠️ 仅限第一方可信应用（如官方 App），不推荐第三方使用

#### 客户端模式（Client Credentials）——服务间通信

用于服务 A → 服务 B 的机器对机器认证，无用户参与

#### 隐式模式（Implicit）——已过时

Token 直接返回前端，易泄露，已被 PKCE 取代

### OAuth2 + OpenID Connect（OIDC）

OAuth2 只负责“授权”，不负责“认证”。  
**OIDC = OAuth2 + ID Token（JWT）** → 可实现“你是谁”+“你能做什么”

### 安全风险与防御

| 风险              | 防御措施                                    |
| ----------------- | ------------------------------------------- |
| 授权码拦截        | 使用 PKCE（Proof Key for Code Exchange）    |
| Token 泄露        | 使用短期 Token + refresh_token，绑定设备/IP |
| 重定向 URL 被篡改 | 严格校验 redirect_uri 白名单                |
| CSRF 攻击         | state 参数校验 + SameSite Cookie            |

---

##防刷机制（Anti-Flooding / Rate Limiting）

### 什么是“刷”？

恶意用户/脚本在短时间内发起大量请求，意图：

- 暴力破解密码
- 刷优惠券/积分
- 压垮服务
- 爬取数据

### 常见防刷策略

#### 限流（Rate Limiting）

- **固定窗口**：每分钟最多 100 次（简单但边界突刺）
- **滑动窗口**：更平滑，如 Redis + Lua 实现
- **令牌桶（Token Bucket）**：匀速处理，允许突发
- **漏桶（Leaky Bucket）**：匀速流出，平滑限流

工具推荐：

- Nginx limit_req
- Redis + Lua（自定义灵活）
- API 网关（如 Kong、APISIX、Spring Cloud Gateway）

#### 用户/设备/IP 维度限制

- 同一 IP 1 分钟内只能发 5 次验证码
- 同一用户 ID 1 天只能领 1 次优惠券

#### 行为验证

- 图形验证码（CAPTCHA）
- 滑块验证（如极验、阿里云盾）
- 设备指纹（Device Fingerprinting）识别模拟器/脚本

#### 异常行为检测

- 同一设备频繁切换账号 → 风控拦截
- 请求频率远超人类操作 → 自动封禁或弹验证码

---

## 防重放攻击（Replay Attack Prevention）

### 什么是重放攻击？

攻击者截获合法请求（如支付请求、登录 Token），在稍后**原封不动重新发送**，以达到重复操作的目的。

示例：

- 截获“转账 100 元”请求 → 重放 10 次 → 转走 1000 元
- 截获登录 JWT → 1 小时后重放 → 仍能登录（若未过期）

### 防御手段

#### 使用一次性 Token / nonce

- 请求中带唯一随机数 `nonce`，服务端记录已使用 nonce，拒绝重复
- 适用于关键操作（如支付、修改密码）

#### 时间戳 + 签名

- 请求带 `timestamp`，服务端校验时间差（如 ±5 分钟），超时拒绝
- 配合签名防篡改（如 HMAC-SHA256）

#### JWT 中嵌入 jti（JWT ID）

- 为每个 JWT 分配唯一 ID，存入 Redis 黑名单，验证是否已使用
- 适合短时效 Token

#### 序列号机制

- 客户端维护递增序列号，服务端记录上一次，拒绝小于等于的请求

---

## 接口安全加固（综合防御体系）

### 传输层安全

强制 HTTPS（TLS 1.2+）  
 HSTS（HTTP Strict Transport Security）头防止降级攻击  
 证书固定（Certificate Pinning）防中间人（App 端常用）

### 请求层安全

签名机制：所有请求带 `sign=md5(param1&param2&key)`，服务端校验  
 参数加密：敏感参数（如手机号、金额）前端加密（RSA/AES）传输  
 敏感接口增加二次验证（短信/邮箱验证码、人脸、指纹）

### 数据层安全

数据库字段加密（如 AES）存储敏感信息（身份证、银行卡）  
 日志脱敏：禁止记录密码、Token、身份证号等  
 权限最小化：接口按角色/权限控制访问（RBAC/ABAC）

### 审计与监控

记录所有关键操作日志（谁、何时、做了什么）  
 异常请求告警（如 1 秒内同一 IP 请求 100 次）  
 实时风控系统（如阿里云风控、自建规则引擎）

### 安全头 & CORS

设置安全响应头：

```http
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
```

严格控制 CORS 白名单，避免 `Access-Control-Allow-Origin: *`

---

## 真实场景案例

### 案例 1：用户登录接口被暴力破解

- **现象**：大量错误密码尝试
- **防御**：
  - 登录失败 5 次 → 锁账号 30 分钟 或 弹验证码
  - IP 维度限流：1 分钟最多 10 次登录请求
  - 使用 JWT + refresh_token，access_token 有效期 15 分钟

### 案例 2：优惠券被脚本刷光

- **现象**：大量请求在 1 秒内领取
- **防御**：
  - 用户维度：每人每天限领 1 张
  - 设备/IP 维度：每分钟限 3 次
  - 关键接口加图形验证码
  - 异步队列削峰 + 库存预扣

### 案例 3：支付请求被重放

- **现象**：同一支付请求被提交多次，导致重复扣款
- **防御**：
  - 请求带唯一订单号 + 服务端幂等校验
  - 支付 Token 一次性使用（jti + Redis 标记）
  - 前端按钮点击后禁用 + loading 状态

---

## 总结对比表

| 安全机制     | 解决什么问题               | 常用技术/方法                      |
| ------------ | -------------------------- | ---------------------------------- |
| JWT          | 无状态身份认证             | HS256/RS256 签名、设置 exp、黑名单 |
| OAuth2       | 第三方授权访问资源         | 授权码模式 + PKCE、OIDC 认证       |
| 防刷         | 防止高频恶意请求           | 限流（令牌桶）、验证码、设备指纹   |
| 防重放       | 防止请求被重复提交         | nonce、timestamp、jti、幂等设计    |
| 接口安全加固 | 综合防御数据泄露/篡改/攻击 | HTTPS、签名、加密、CSP、审计日志   |
