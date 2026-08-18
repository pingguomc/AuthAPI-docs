# 端点：/management/console

管理后台端点。本端点需要独立的**后台鉴权**，与主站 `/management` 及用户会话完全隔离。

后台鉴权机制参考 [管理后台](../../index.md#管理后台)：仅已登录主站 **Admin** 角色的用户可进行后台登录；后台采用独立账户体系（`console_admins`），分为 `admin` 与 `super_admin` 两个级别，通过二次登录输入系统生成的分发密码进入后台。

后台操作一律写入[后台审计日志](../../SQL.md#后台审计日志-console_audit_logs)（含操作者、IP、浏览器信息）。

## 鉴权

后台使用独立会话，凭据通过 `Set-Cookie` 下发（`console_sid`），`HttpOnly`。

### POST /management/console/auth/login

后台登录（Admin 二步验证：先已登录主站为 Admin，再以本接口输入后台密码）。

**请求体**：

```json5
{
  "password": "系统分发的强密码"
}
```

**响应**：成功返回 `200`，设置后台会话 Cookie；无响应体，或返回：

```json5
{
  "role": "super_admin",
  "username": "admin1"
}
```

**错误**：
- 密码错误或主站会话非 Admin：`401`，`error` 为 `InvalidCredentials`。
- 该后台账户已被禁用：`403`，`error` 为 `ConsoleAdminDisabled`。

### POST /management/console/auth/logout

登出后台。清除后台会话 Cookie。

**响应**：`204`。

### GET /management/console/auth/me

获取当前后台登录信息。

**响应**：`200`：

```json5
{
  "username": "admin1",
  "role": "super_admin",
  "sessionId": "c_01H..."
}
```

未登录返回 `401`，`error` 为 `Unauthorized`。

## 角色 / 权限授予

仅 `super_admin` 可执行本组操作（`admin` 返回 `403`）。

### GET /management/console/users

后台可管理的用户列表（供授予操作选择目标）。

**请求参数**：同 [GET /management/users](../index.md#get-managementusers)。

**响应**：`200`，默认返回 `id`、`username`、`email`、`role`、`permissions`（用户已有权限节点）。

### POST /management/console/users/{userId}/roles

变更用户**系统角色**（`user` / `helper` / `moderator` / `admin`）。

**请求体**：

```json5
{
  "role": "moderator",
  "reason": "任命版主" // 可选
}
```

**响应**：`201`：

```json5
{
  "id": "rg_01H...",
  "userId": "be081798-...",
  "previousRole": "user",
  "role": "moderator",
  "operatorId": "be081798-...",
  "createdAt": "2026-08-08T10:30:00Z"
}
```

**备注**：
- `role` 不在四者内返回 `400`，`error` 为 `InvalidRole`。
- 变更成功会**清除该用户全部会话**以保证即时生效。
- 存在锁死风险（如唯一 SuperAdmin 将某人提权后无法管理），需谨慎并写入审计。

### GET /management/console/permission-grants

列出所有额外的[用户权限节点授予](../../SQL.md#用户权限节点-user_permissions)（超出角色内建集合的部分）。

**查询参数**：`userId`（可选）。

**响应**：`200`：

```json5
{
  "items": [
    { "userId": "be081798-...", "permissionNode": "management.users", "grantedBy": "be081798-...", "grantedAt": "..." }
    // ...
  ]
}
```

### POST /management/console/permission-grants

授予用户一个额外权限节点。

**请求体**：

```json5
{
  "userId": "be081798-...",
  "permissionNode": "management.users"
}
```

**响应**：`201`。

**备注**：已存在同一 `(userId, permissionNode)` 返回 `409`，`error` 为 `PermissionAlreadyGranted`。

### DELETE /management/console/permission-grants/{userId}/{node}

撤销某个用户的一个额外权限节点（`node` 为节点名，如 `management.users`）。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `PermissionGrantNotFound`。

## 配置热重载

### POST /management/console/reload

热重载全部配置文件（对应后端命令系统的 `/reload`，见 [后端命令系统](../../backend.md)）。

**请求体**：无。

**响应**：`200`：

```json5
{
  "reloaded": [
    "permissions.toml",
    "rate-limit.toml",
    "OIDC.toml"
  ],
  "reloadedAt": "2026-08-08T10:30:00Z"
}
```

任意文件解析失败返回 `400`，`error` 为 `ConfigParseError`，此时**不应用**任何改动。

## 后台审计日志

### GET /management/console/audit-logs

后台审计日志查询（**只读**，见 [console_audit_logs](../../SQL.md#后台审计日志-console_audit_logs)）。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `adminId` | string | 后台操作者 |
| `action` | string | 动作，前缀匹配 |
| `result` | string | `success` / `denied` |
| `from` | string | 时间下界，ISO 8601 UTC |
| `to` | string | 时间上界，ISO 8601 UTC |
| `page` | int | 页码，默认 1 |
| `pageSize` | int | 每页条数，默认 20，上限 100 |

固定按 `createdAt` 倒序。

**响应**：`200`：

```json5
{
  "items": [
    {
      "id": "clog_01H...",
      "adminId": "be081798-...",
      "action": "console.role_grant",
      "result": "success",
      "ip": "203.0.113.1",
      "userAgent": "Mozilla/5.0 ...",
      "payload": { "targetUserId": "...", "role": "moderator" },
      "createdAt": "2026-08-08T10:30:00Z"
    }
    // ...
  ],
  "page": 1,
  "pageSize": 20,
  "total": 120
}
```

**action 取值**：

| action | 说明 |
|--------|------|
| `console.login` / `console.logout` | 后台登录 / 登出 |
| `console.role_change` | 变更用户系统角色 |
| `console.permission_grant` / `console.permission_revoke` | 授予 / 撤销权限节点 |
| `console.reload` | 配置热重载 |

主站操作审计见 [主站审计日志](../index.md#审计日志)。