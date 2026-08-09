# 端点：/captcha

人机验证相关端点。本端点 **不需要身份验证**。

人机验证 token 不通过请求体传递，而是通过请求头下发,作用于**其他端点**(如注册、登录、发验证码等),
详见[人机验证请求头](#人机验证请求头)。

## 目录

- [端点:/captcha](#端点captcha)
    - [GET /captcha/config](#get-captchaconfig)
- [人机验证请求头](#人机验证请求头)
- [受保护的动作](#受保护的动作)
- [开关约定](#开关约定)
- [错误码](#错误码)
- [后端处理流程](#后端处理流程)
- [测试密钥](#测试密钥)

## GET /captcha/config

获得人机验证配置。前端启动时或进入受保护页面前调用,用于决定是否加载 SDK、加载哪一家。

**请求**:请求体为空。

**响应**:成功返回 HTTP 状态码 `200`。

启用时:
```json5
{
  "enabled": true,                  // 是否启用人机验证(全局开关)
  "configVersion": 42,              // 配置版本号,每次开关变更递增
  "provider": "turnstile",          // turnstile | hcaptcha | recaptcha | geetest
  "siteKey": "0x4AAAAAAA-xxxx",     // 公开密钥,可下发
  "scriptUrl": "https://challenges.cloudflare.com/turnstile/v0/api.js", // SDK 地址,由后端下发
  "options": {                      // 透传给 widget 的可选项,可为 null
    "theme": "auto",
    "size": "normal"
  },
  "actions": {                      // 各动作的开关,键为动作标识
    "register": { "enabled": true,  "mode": "always" },
    "login": { "enabled": true,  "mode": "always" },
    "email-code": { "enabled": true,  "mode": "always" },
    "change-password": { "enabled": false, "mode": "onDemand" }
  }
}
```

全局关闭时(前端必须能处理此精简形态):
```json5
{
  "enabled": false,
  "configVersion": 43,
  "actions": {}
}
```

**字段说明**:

|                 字段 |    类型    | 说明                                        |
|-------------------:|:--------:|-------------------------------------------|
|          `enabled` | boolean  | 全局开关。为 `false` 时其余字段可为 null 或省略           |
|    `configVersion` | integer  | 配置版本号,用于开关切换期间的竞态处理                       |
|         `provider` |  string  | 当前使用的 provider,`enabled` 为 `false` 时可为 null |
|          `siteKey` |  string  | 公开密钥。**secret 绝不下发**                      |
|        `scriptUrl` |  string  | SDK 脚本地址,由后端下发以便切换 CDN 或 provider         |
|          `options` |  object  | 透传给 widget 的渲染选项,可为 null                  |
|          `actions` |  object  | 动作开关表。键为动作标识,见[受保护的动作](#受保护的动作)           |
|  `actions[*].enabled` | boolean | 该动作是否需要人机验证                                |
|     `actions[*].mode` | string  | `always`:每次都需要;`onDemand`:平时不需要,后端按风控临时索要 |

**备注**:
* 该端点响应可被缓存,建议后端设置 `Cache-Control: max-age=60` 与 `ETag`。
* `provider` 与 `siteKey` 一律由本端点下发,前端**禁止硬编码**。
* 将来切换 provider 或新增国产方案(极验、易盾等)仅需修改后端配置,前端零发布。

## 人机验证请求头

需要人机验证的端点,通过以下**请求头**携带 token,请求体保持不变。

```http
POST /user/login
Content-Type: application/json; charset=utf-8
X-Captcha-Provider: turnstile
X-Captcha-Token: 0.aBcDeF...
X-Captcha-Action: login
X-Captcha-Config-Version: 42

{
  "email": "user@example.com",
  "password": "Abc123"
}
```

|                        请求头 |     必填     | 说明                                 |
|---------------------------:|:----------:|------------------------------------|
|          `X-Captcha-Token` | 该动作开关开启时必填 | widget 产出的一次性 token                |
|       `X-Captcha-Provider` |     建议     | 便于 provider 切换过渡期内后端同时接受两家 token   |
|         `X-Captcha-Action` |     建议     | 动作标识,后端校验其与端点匹配,防止跨端点复用 token      |
| `X-Captcha-Config-Version` |     可选     | 前端当前所持配置版本,便于后端观察灰度覆盖率             |

**备注**:
* 不设独立的 `/captcha/verify` 端点。独立端点会产出可重放的 ticket,需额外管理一次性、过期与会话绑定,得不偿失。
* token 为一次性,提交失败后必须重新获取,不可复用。

## 受保护的动作

|                  动作标识 | 对应端点                                                           |
|----------------------:|----------------------------------------------------------------|
|            `register` | [POST /user/register](./user.md#post-userregister)             |
|               `login` | [POST /user/login](./user.md#post-userlogin)                   |
| `email-code-register` | [POST /email/code/register](./email.md#post-emailcoderegister) |
|    `email-code-login` | [POST /email/code/login](./email.md#post-emailcodelogin)       |

**备注**:
* `/email/code/*` 是最需要保护的一组端点(邮件轰炸)，建议始终开启。
* OIDC 相关端点为 302 重定向流程，不适用人机验证。

## 开关约定

开关分三级,任一级关闭则该动作不需要人机验证:

```
enabled = false                     → 整站关闭,所有动作放行
   ↓ true
actions[x].enabled = false          → 该动作关闭,放行
   ↓ true
mode = "onDemand" 且风控未命中        → 本次放行
```

**关闭状态下的行为(必须严格遵守)**:

|  角色 | 行为                                              |
|----:|-------------------------------------------------|
| 前端  | 不加载 SDK、不渲染 widget、不发送 `X-Captcha-*` 请求头        |
| 后端  | 完全跳过校验;即使请求携带了 token 也直接忽略             |
| 后端  | 不因人机验证关闭而放宽限流、登录失败次数等其他风控               |

**开关必须支持运行时热更新**,不允许需要重启服务或前端发版。
provider 故障或大面积误杀时,运维需能在 1 分钟内将 `enabled` 置为 `false` 摘除全站。

**开关切换竞态**:后端刚开启开关而前端仍持旧配置时,业务请求会缺 token。
此时后端返回 `CAPTCHA_MISSING`,前端**必须**重新拉取配置、渲染 widget 并自动重试一次,
且**不得清空用户已填写的表单内容**。反向竞态(后端已关闭、前端仍发 token)由上表"忽略不报错"兜底。

## 错误码

沿用[统一错误信息格式](../index.md#错误信息格式)。

|  HTTP |                       error | 含义                     | 前端处理                 |
|------:|----------------------------:|------------------------|----------------------|
| `400` |          `CAPTCHA_MISSING`  | 该动作需要人机验证但未携带 token    | 重新拉取配置 → 渲染 widget → 自动重试 |
| `400` |          `CAPTCHA_EXPIRED`  | token 已过期(Turnstile 为 300 秒) | 重置 widget,保留表单数据后重试  |
| `400` |          `CAPTCHA_INVALID`  | provider 判定失败,或 token 已被使用 | 重置 widget 后重试        |
| `400` |  `CAPTCHA_ACTION_MISMATCH`  | token 的动作标识与端点不匹配      | 重新验证                 |
| `403` |         `CAPTCHA_REQUIRED`  | `onDemand` 模式下后端临时索要验证 | 弹出 widget,完成后自动重试原请求 |
| `403` |    `CAPTCHA_SCORE_TOO_LOW`  | 评分型 provider 分数不足(预留)  | 提示用户或降级为交互式挑战        |
| `503` | `CAPTCHA_SERVICE_UNAVAILABLE` | provider 接口故障且策略为 fail-closed | 提示稍后重试               |

**响应示例**:
```json5
{
  "error": "CAPTCHA_EXPIRED",
  "errorMessage": "人机验证已过期,请重新验证",
  "cause": "timeout-or-duplicate"    // 可选,provider 原始错误码
}
```

**备注**:任何人机验证错误都**不得导致用户已填内容丢失**。

## 后端处理流程

以受保护端点为例:

1. 读取配置(建议本地缓存 5 秒),若 `enabled` 为 `false`,跳过校验,进入业务逻辑
2. 若 `actions[action].enabled` 为 `false`,跳过校验
3. 若 `mode` 为 `onDemand` 且风控未命中,跳过校验;命中则返回 `403 CAPTCHA_REQUIRED`
4. 读取 `X-Captcha-Token`,为空则返回 `400 CAPTCHA_MISSING`
5. 调用 provider 的 siteverify 接口,超时设为 2~3 秒
6. 校验 `success`、`hostname`(须与本站域名一致)、`action`(须与端点匹配)
7. 校验失败则按[错误码](#错误码)映射返回;调用异常按失败策略处理
8. 通过后进入业务逻辑(如登录、发验证码)

**Turnstile siteverify 请求**:
```json5
{
  "secret": "<从环境变量读取>",
  "response": "<前端提交的 token>",
  "remoteip": "<客户端真实 IP>",
  "idempotencyKey": "<token 的 SHA256,超时重试时复用>"
}
```

**注意事项**:
* `secret` 只进环境变量或密钥管理服务,**不入库、不下发、不进日志**。
* token TTL 为 300 秒,**不得缓存验证结果**,必须在业务处理时刻实时验证。
* siteverify 超时重试必须复用同一 `idempotencyKey`(30 分钟内有效),否则会被判为"token 已使用"。
* `remoteip` 需正确穿透反向代理(取 `X-Forwarded-For` 最左可信段)。
* 人机验证**不替代风控**。登录失败次数限制、IP 与账号频控仍须独立生效,且不随人机验证开关关闭。
* 开关变更需记录审计日志(操作人、时间、变更前后值)。
* 配置中心不可用时,使用上一次的有效配置,而非退化为"全关"或"全开"。

**失败策略**:provider 接口故障时,默认采用 **fail-open**(放行)并收紧限流阈值,
避免故障期间全站无法登录。该策略可通过配置切换为 fail-closed,切换时须打告警日志。

## 测试密钥

CI 与联调环境使用官方测试密钥,不请求真实服务。

|           场景 | siteKey                    | secret                                |
|-------------:|----------------------------|---------------------------------------|
| Turnstile 总是通过 | `1x00000000000000000000AA` | `1x0000000000000000000000000000000AA` |
| Turnstile 总是失败 | `2x00000000000000000000AB` | `2x0000000000000000000000000000000AA` |
| Turnstile 强制交互 | `3x00000000000000000000FF` | —                                     |

hCaptcha 与 reCAPTCHA 的官方测试密钥待接入时补充。