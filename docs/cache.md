# 缓存（Cache）说明

本文档列出系统所有**存于内存缓存**（如 Redis）的数据。这些数据**不落数据库**，丢失后可自行重建或自然过期。缓存键统一使用冒号分隔的前缀命名。

## 主站会话（session）

| 键 | 值 | TTL | 说明 |
|----|-----|-----|------|
| `session:{sid}` | `{ userId, 过期时间, ... }` | 跟随 Cookie `Max-Age`（如 86400s） | 主站登录会话。`sid` 通过 `Set-Cookie` 下发，会话对象存缓存 |

会话失效包含：显式登出、改密码清除全部会话、封禁/角色变更清会话等。**不落库**。

## 后台会话（JWT）

| 键 | 内容 | TTL | 说明 |
|----|-----|-----|------|
| `console_session:{jti}` | `{ adminUserId, role, 过期时间 }` | 如 3600s | 后台登录会话。`jti` 为 JWT 唯一标识，会话对象存缓存 |

后台 JWT：登录时签发，签名为后台密钥，`jti` 在缓存登记。请求经 `Authorization: Bearer {JWT}` 携带，后端解析 JWT 并由缓存校验 `jti` 有效（未登出/未过期）。

## Yggdrasil 令牌

| 键 | 内容 | TTL | 说明 |
|----|-----|-----|------|
| `ygg_token:{accessToken}` | `{ clientToken, authRecordId, profileId, 颁发时间, 过期时间, invalidUntil? }` | 令牌过期（如 15 天） | `accessToken` 关联的令牌对象。状态由缓存派生 |

## 邮箱验证码

| 键 | 内容 | TTL | 说明 |
|----|-----|-----|------|
| `email_code:{email}:{purpose}` | `{ code, 尝试次数 }` | 如 300s | 邮箱验证码。`purpose` 区分注册/登录/改密/改邮箱 |

## OIDC 流程

| 键 | 内容 | TTL | 说明 |
|----|-----|-----|------|
| `oidc_state:{state}` | `{ codeVerifier, action, redirectUri, ... }` | 如 600s | OIDC 授权流程上下文，`state` 防 CSRF |
| `oidc_nonce:{nonce}` | `{ 已使用 }` | 一次性 | 可选，防重放 |

## 人机验证

| 键 | 内容 | TTL | 说明 |
|----|-----|-----|------|
| `captcha:{token}` | `{ 已使用 }` | token TTL（Turnstile 300s） | 记录一次性人机验证 token，防止重放 |

## 限流计数

| 键 | 内容 | TTL | 说明 |
|----|-----|-----|------|
| `ratelimit:{维度}:{标识}` | 请求计数 | 按限速窗口 | 限流计数，维度如 `ip`、`userId`、`email`、`ip+email` 等 |

## 其他

如有临时生成的随机数、一次性凭据等，可复用 `cache:` 通用前缀；超期自动清除。