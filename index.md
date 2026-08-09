# Auth API 标准规范文档

前后端通用。

* [user 端点](./docs/user.md)
* [email 端点](./docs/email.md)
* [admin 端点](./docs/admin.md)

具体的 back 和 front 文档仅供参考，可不按要求。但 API 接口必须按要求编写，以便实现前后端随意适配。

## 技术约定

* 遵循 `RESTful API` 技术规范。
* 本文中字符编码一律使用 UTF-8。
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
