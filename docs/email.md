# 端点：/email

本端点需要身份验证的采用 [Cookie HttpOnly 会话](../index.md#cookie-格式)，在端点后标 (cookie身份验证)。

## 端点：/email/code

验证码端点。

### POST /email/code/register (人机验证)

[注册](./user.md#post-userregister)验证码。

**请求**：
```json5
{
  "email": "user@example.com"
}
```

**响应**：成功则返回HTTP状态码 `200`。

备注：需校验邮箱是否已被注册，以决定是否发送验证码。若已被另一账户注册，本次不发验证码，返回 `409`,`error` 为 `EmailAlreadyRegistered`。

### POST /email/code/login (人机验证) 

[登录](./user.md#post-userlogin)验证码。

**请求**：
```json5
{ 
  "email": "user@example.com"
}
```

**响应**：
成功则返回HTTP状态码 `200`。

备注：需校验邮箱是否已被注册。若未注册，返回 `404`,`error` 为 `EmailNotRegistered`,且不发验证码。

### POST /email/code/change-password (Cookie身份验证)

[使用邮箱验证码更改密码](./user.md#post-userchange-password-cookie身份验证)验证码。

**请求**：请求体为空，凭据通过 Cookie 传递

**响应**：
成功则返回HTTP状态码 `200`。

**备注**：若用户未绑定邮箱，此接口返回 `400`。

### POST /email/code/set-email (Cookie身份验证)

[设置或更改邮件](./user.md#put-useremail-cookie身份验证)验证码。

**请求**：
```json5
{ 
  "email": "user@example.com"
}
```

**响应**：
成功则返回HTTP状态码 `200`。

备注：无论当前用户是否已有邮箱，均使用此端点（首次绑定 / 更换均适用）。需校验邮箱是否已被其他用户注册。