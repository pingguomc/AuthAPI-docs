# Auth API 标准规范文档

前后端通用。

## 本目录索引

### 主端点

- [user 端点](EP-user.md)
- [email 端点](EP-email.md)
- [management 端点](management/index.md)
- [captcha 端点](EP-captcha.md)
- [错误码速查表](./error.md)
- [数据库表结构（SQL）](./SQL.md)
- [速率限制](./ratelimit.md)

### Yggdrasil 相关

- [Yggdrasil 入口](./yggdrasil/index.md)
- [材质和扩展 API](yggdrasil/EP-api.md)
- [authserver（用户部分）](yggdrasil/EP-authserver.md)
- [sessionserver（会话与角色）](yggdrasil/EP-sessionserver.md)
- [签名密钥对](./yggdrasil/signature.md)

## 技术约定

若无特别说明，遵循以下规则。

* 遵循 `RESTful API` 技术规范。
* 本文中字符编码一律使用 UTF-8。
* 请求与响应均为 JSON 格式。
* `Content-Type` 均为 `application/json; charset=utf-8`。
* 统一使用 ISO 8601 格式的 UTC 字符串表示时间。
* 密码使用 Bcrypt 加密存储。

### 错误信息格式

Http 状态码按通用约定返回。

```json5
{
  "error":"错误的简要描述（机器可读）",
  "errorMessage":"错误的详细信息（人类可读）",
  "cause":"该错误的原因（可选）"
}
```

`error` 使用大驼峰命名法。`errorMessage` 使用中文描述。

### Cookie 格式

后端通过 `Set-Cookie` 响应头下发会话凭据。

|    属性    |     值     | 说明          |
|:--------:|:---------:|-------------|
|   Name   |   `sid`   | 会话标识符       |
| HttpOnly |   true    | 前端 JS 不可读   |
|  Secure  |   true    | 仅 HTTPS 传输  |
| SameSite |    Lax    | 防 CSRF      |
|   Path   |     /     |             |
| Max-Age  | 86400（示例） | 24 小时过期（示例） |

## 模型

### 系统角色、权限模型、用户身份

系统角色依次为：`User` 用户、`Helper` 协管、`Moderator` 版主、`Admin` 管理员。  
系统角色层级为 User < Helper < Moderator < Admin。

权限为权限节点式，凡是需要权限的端点都在端点中标记出来，权限检查以是否拥有权限节点为准。

用户身份需要在配置文件中定义，需指明 id、展示名（用作前缀或者后缀）、权限节点。用户身份和系统角色不绑定。  
权限节点均从配置文件动态读取，不写死具体权限。

#### 管理后台

其中一些全局性的或者危险的操作，则需要使用管理后台，只有 Admin 用户可以使用。  
管理后台采用独立的鉴权机制，需要已登录主站的 Admin 角色进行二次登录，输入强密码。此密码为直接生成，不是由管理员自行设置。  
第一个管理员如何设置参考后端命令系统。

管理后台的权限不使用权限节点制度，而是分为 Admin 和 SuperAdmin 两个管理后台专属级别，和主站独立。 SuperAdmin 最多设置两名。  
后台的权限要求由后台端点规定。

### 审计日志

对于主站的全部需要权限节点的操作，都计入主站审计日志。需同步记录登录IP。

对于后台的全部操作，则计入后台审计日志。需同步记录登录IP、浏览器信息等。

## 根目录数据端点

`GET /`

**请求**：无请求体和请求头。

**响应**：添加响应头 `X-Authlib-Injector-API-Location` ，值为 `/yggdrasil/`。
```json5
{
  "status": "normal", // 若为 normal 则前端正常提供服务，若为 maintenance，前端停止一切服务并展示维护页。
  "feature": {  //功能
    "email_register": true, //当前是否开启邮箱注册
    "email_login": true, //当前是否开启邮箱登录
    "find_oidc": true, //当前是否开启 OIDC （若开启，前端再访问/user/oidc/providers）
    "find_captcha": true //当前是否开启人机验证 （若开启，前端再访问/captcha/config）
  },
  "motd": "Message of the Day"
}
```

## 其他
