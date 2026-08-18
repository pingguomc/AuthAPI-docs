# Auth API 标准规范文档

前后端通用。

## 本目录索引

### 主端点

- [user 端点](EP-user.md)
- [email 端点](EP-email.md)
- [management 端点](management/index.md)
- [management console 后台端点](management/console/index.md)
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

### 系统角色、身份组、权限模型

采用**并行双模型**：

- **系统角色（role）**：`User` 用户、`Helper` 协管、`Moderator` 版主、`Admin` 管理员。层级 User < Helper < Moderator < Admin。用户必有一个，权限节点表相对固定（内建）。
- **身份组（identity group）**：相当于自定义角色，一名用户可属于**多个**身份组（也可不属任何组）。身份组的元数据（id、名称）存库，权限节点列表由配置文件定义。

权限为粒度式，凡是需要权限的端点都在端点中标记出来，并标注默认最低系统角色。  
实际权限 = **系统角色内建节点 ∪ 该用户所属全部身份组节点**（并集），任一来源拥有即放行。  
权限节点均从配置文件动态读取，不写死具体权限。

#### 管理后台

其中一些全局性的或者危险的操作，则需要使用管理后台，仅系统角色为 `Admin` 的用户可使用。  
每个 Admin 用户有**唯一**后台账户；后台采用独立鉴权：登录需校验主站 Cookie（确认已登录且为 Admin）+ 后台密码（由系统直接生成），签发后台 JWT。后台请求需同时携带主站 Cookie 与后台 JWT。  
后台操作一律计入后台审计日志；主站需要权限节点的操作计入主站审计日志。

#### 用户展示

用户前端展示格式为 `[前缀]username[身份组][系统角色]`，不再使用 `display_name`（保留字段，一律展示 `username`）。前缀由用户自行修改。

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
