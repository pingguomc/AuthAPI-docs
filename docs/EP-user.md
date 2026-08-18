# 端点：/user

本端点需要身份验证的采用 [Cookie HttpOnly 会话](./index.md#cookie-格式)，在端点后标 (cookie身份验证)。

## 目录

- [端点：/user](#端点user)
   - [POST /user/register](#post-userregister)
   - [POST /user/login](#post-userlogin)
   - [GET /user/me (Cookie身份验证)](#get-userme-cookie身份验证)
   - [PUT /user/prefix (Cookie身份验证)](#put-userprefix-cookie身份验证)
   - [POST /user/change-password (Cookie身份验证)](#post-userchange-password-cookie身份验证)
   - [PUT /user/email (Cookie身份验证)](#put-useremail-cookie身份验证)
   - [POST /user/logout (Cookie身份验证)](#post-userlogout-cookie身份验证)
- [端点：/user/oidc](#端点useroidc)
   - [GET /user/oidc/providers](#get-useroidcproviders)
   - [GET /user/oidc/{providerId}/authorize](#get-useroidcprovideridauthorize)
   - [GET /user/oidc/{providerId}/bind (Cookie身份验证)](#get-useroidcprovideridbind-cookie身份验证)
   - [GET /user/oidc/{providerId}/callback](#get-useroidcprovideridcallback)
   - [DELETE /user/oidc/{providerId} (Cookie身份验证)](#delete-useroidcproviderid-cookie身份验证)

## POST /user/register

使用邮件注册。

**请求**：
```json5
{ 
  "email": "user@example.com",
  "password": "Abc123",
  "emailCode": "123456",
  "displayName": "显示的用户名"
}
```

**响应**：成功返回HTTP状态码 `201`：
```json5
{
  "createdAt": "2026-08-08T10:30:00Z" //创建时间
}
```

**备注**：需校验邮箱是否已被注册。

## POST /user/login

使用密码或邮箱验证码登录。

**请求**：
```json5
{
  "email": "user@example.com",
  "password": "Abc123"
}
```
或
```json5
{
  "email": "user@example.com",
  "emailCode": "123456"
}
```

**响应**：成功返回HTTP状态码 `200`，通过 `Set-Cookie` 响应头下发会话凭据，无响应体。

## GET /user/me (Cookie身份验证)

获得用户信息。

**请求**：请求体为空，凭据通过 Cookie 传递

**响应**：
```json5
{
  "userId": "be081dbc-3de9-4138-9e13-3cbc5439dd4a", // 随机示例
  "role": "user", // 系统角色（非空），取值参考 ./index.md 系统角色与权限模型
  "prefix": "[前缀]", // 显示前缀，可为空
  "identityGroups": [ // 所属身份组，可为空
    { "id": "g_01H...", "name": "groupA", "displayName": "Group A" }
  ],
  "username": "user_be08...", // 对外展示一律用 username
  "email": "绑定的邮箱", // 可能为空字符串
  "hasPassword": false, // 是否已设置密码
  "bindingOIDC": ["google","github"] // 内容为 providerID，可能为空
}
```

## PUT /user/prefix (Cookie身份验证)

修改显示前缀（用户自行修改，`prefix` 可设为空字符串清除）。

**请求**：
```json5
{
  "prefix": "[新前缀]"
}
```

**响应**：成功返回 `204`，无响应体。

**后端处理**：更新当前用户的 `prefix` 字段。

**备注**：前缀仅作展示，不影响权限。

## POST /user/change-password (Cookie身份验证)

使用旧密码或者邮箱验证码更改密码。

**请求**：
```json5
{
  "oldPassword": "Abc123",
  "newPassword": "NewPass123"
}
```
或
```json5
{
  "emailCode": "123456",
  "newPassword": "NewPass123"
}
```

**后端处理**：
1. 根据请求体类型：
    * 若传了 `oldPassword`：校验旧密码是否正确
    * 若传了 `emailCode`：校验 emailCode 是否正确且未过期
2. 数据库更新密码
3. 清除该用户所有已有 session
4. 若使用了 emailCode，清除该验证码

**响应**：成功返回HTTP状态码 `204`，响应头 `Set-Cookie` 将 `sid` 设为过期，无响应体。

**备注**：仅限登录的用户更改密码所用。忘记密码无法登录者可通过邮箱验证码或OIDC登录。

## PUT /user/email (Cookie身份验证)

设置或更改邮件。

**请求**：
```json5
{
  "email": "newemail@example.com",
  "emailCode": "123456"
}
```

**后端处理**：
1. 校验 emailCode 是否正确且未过期
2. 校验 `email`(请求中的新邮箱)与发验证码时使用的邮箱一致
3. 数据库更新邮箱
4. 清除该验证码

**响应**：成功返回 `200`
```json5
{
  "email": "newemail@example.com"
}
```

## POST /user/logout (Cookie身份验证)

登出。

**请求**：请求体为空，凭据通过 Cookie 传递

**后端处理**：清除 session。

**响应**：成功返回 `204`，响应头 `Set-Cookie` 将 `sid` 设为过期，无响应体。

## 端点：/user/oidc

OIDC 相关的用户操作端点。

### GET /user/oidc/providers

获得授权服务器列表。

**请求**：请求体为空

**响应**：
```json5
{
  "enabled": true, // 是否允许 OIDC  
  "providers": [
    {
      "providerId": "google",
      "displayName": "Google",         // 用于按钮展示的名称，可为 null
      "iconUrl": "https://cdn.example.com/icons/google.svg", // 图标地址，可为 null
    },
    {
      "providerId": "github",
      "displayName": "GitHub",
      "iconUrl": "https://cdn.example.com/icons/github.svg",
    }
  ]
}
```

### GET /user/oidc/{providerId}/authorize

通过指定的授权服务器登录或注册。

**请求**： 请求体为空
可选查询参数：

| 参数           | 说明                                         |
|--------------|--------------------------------------------|
| redirect_uri | 登录成功后跳回的前端页面地址（可选，须在白名单内；未传时使用后端配置的默认跳转地址） |

**后端处理**：
1. 生成随机 `state`，存入 session
2. 生成随机 `code_verifier`（43~128 字符），存入 session
3. 计算 `code_challenge = base64url(sha256(code_verifier))`
4. 在 session 中标记 `action = "login"`
5. 若前端传了 `redirect_uri`，校验其必须命中 `config/OIDC.toml` 的 `allowed_redirect_uris` 白名单（否则 403 拒绝发起流程）；将生效的跳转地址存入 session 作为回调后的跳转目标。未传时使用 `default_redirect_uri`（未配置则回调回退到根路径 `/`）
   - 白名单匹配比较 `scheme + host:port + path`，**忽略 query 与 fragment**，因此可在 `redirect_uri` 中携带 `after` 等查询参数（例如传入 `http://localhost:5173/oidc/callback?after=/dashboard` 时，配置 `http://localhost:5173/oidc/callback` 即可命中）
6. 构造 Provider 授权 URL，附加 `code_challenge` 和 `code_challenge_method=S256`，302 重定向

**响应**：若成功，则 `302` 重定向至指定的 Provider （ Authorization Server ）的授权页。前端应通过新窗口或直接跳转的方式访问此端点。

### GET /user/oidc/{providerId}/bind (Cookie身份验证)

在已登录的情况下绑定新授权服务商。

**请求**：凭据通过 Cookie 传递，请求体为空。
可选查询参数：

| 参数             | 说明                        |
|----------------|---------------------------|
| `redirect_uri` | 绑定成功后跳回的前端页面地址（可选，须在白名单内） |

**后端处理**：
1. 校验用户已登录
2. 生成随机 `state`，存入 session
3. 生成随机 `code_verifier`（43~128 字符），存入 session
4. 计算 `code_challenge = base64url(sha256(code_verifier))`
5. 在 session 中标记 `action = "bind"`
6. 若前端传了 `redirect_uri`，校验其必须命中 `allowed_redirect_uris` 白名单（否则 403 拒绝）；将生效的跳转地址存入 session 作为回调后的跳转目标。未传时使用 `default_redirect_uri`（未配置则回退到根路径 `/`）。匹配规则同 authorize（比较 `scheme + host:port + path`，忽略 query）
7. 构造 Provider 授权 URL，附加 `code_challenge` 和 `code_challenge_method=S256`，302 重定向

**响应**：与 `/authorize` 相同，`302` 重定向至 Provider 授权页。

### GET /user/oidc/{providerId}/callback

此端点由 Provider 在用户授权后自动调用，**前端无需直接访问**。  
登录与绑定共用此回调地址，后端通过 session 中的 `action` 标记区分。  
使用 `Authorization Code Flow` + `PKCE` 流程。

**请求（Provider 传递）**：使用 URL **查询** 参数

|         参数          | 说明                             |
|:-------------------:|--------------------------------|
|       `code`        | 授权码，一次性，后端立即用 token 端点换取 token |
|       `state`       | 后端发起授权时生成的，用于防 CSRF，必须校验       |
|       `error`       | 失败时出现，如 `access_denied`        |
| `error_description` | 失败时的人类可读描述                     |

**后端处理**：
1. 校验 `state` 是否匹配
2. 若存在 `error` 参数，直接 302 重定向到前端，附带错误信息
3. 用 `code` + `code_verifier`（从 session 取出）调 Provider 的 `/token` 端点
4. 验证 `id_token`（签名、iss、aud、exp）
5. 可选：调 `/userinfo` 获取更多 claims
6. 根据 session 中的 `action` 决定：
    - `login`：创建或匹配本地用户，建立本地 session（`Set-Cookie`）
    - `bind`：校验用户已登录，将 Provider 账号关联到当前用户（若该 Provider 账号已被其他用户绑定，返回错误；若当前用户已绑定该 Provider，视为幂等，直接成功）
7. 从 session 取出生效的跳转地址（授权时存入：传入且命中白名单 → 使用它；未传 → 使用 `default_redirect_uri`；均无 → 根路径 `/`），302 重定向至该地址。

> **说明**：跳转地址优先取授权时存入 session 的值。若 `state` 缺失/过期（流程上下文不可还原），则直接回退到 `default_redirect_uri`（未配置则根路径 `/`）。

**响应（重定向到前端时）**：
成功：302 跳转到前端回调页。登录时通过 `Set-Cookie` 建立会话；绑定时无需额外操作。
失败：302 跳转到前端回调页，URL 附加以下查询参数：

|       参数       | 说明                     |
|:--------------:|------------------------|
|    `status`    | 值为 `error`             |
|    `error`     | 错误码                    |
| `errorMessage` | 人类可读的错误描述（URL encoded） |


**备注**：当用户因处于封禁状态而无法登录时，`error` 查询参数的取值固定为 `UserBanned`。此时后端不会建立会话，前端应据此展示封禁提示（例如解析 `errorMessage` 展示给用户）。

### DELETE /user/oidc/{providerId}  (Cookie身份验证)

在登录的情况下，删除指定的授权服务商。

**请求**：凭据通过 Cookie 传递，请求体为空。

**后端处理**：
1. 校验用户已登录
2. 校验该 Provider 确实已绑定到当前用户
3. 校验解绑后用户仍有其他登录方式（邮箱或至少一个其他 OIDC 绑定）

**响应**：成功返回 `200`：
```json5
{
   "bindingOIDC": ["google"]  // 解绑后剩余的绑定列表
}
```