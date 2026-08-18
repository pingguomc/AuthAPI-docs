# 端点：/management/console

管理后台端点。仅系统角色为 `admin` 的用户可使用（经后台独立鉴权）。

## 鉴权机制

后台采用**独立鉴权**，与主站会话隔离：

- 一个系统角色为 `admin` 的用户，后台账户有且唯一（见 [console_admins](../../SQL.md#后台账户-console_admins)），密码由后台生成/重置。
- **登录**：校验主站 Cookie（确认已登录且为 Admin）+ 输入后台密码，通过后签发后台 JWT。
- **请求校验**：后台业务接口**同时校验**主站 Cookie（确认当前用户）和请求头携带的后台 JWT（如 `Authorization: Bearer <jwt>`），两者都有效才放行。
- 后台操作一律写入[后台审计日志](../../SQL.md#后台审计日志-console_audit_logs)（含操作者、IP、浏览器信息）。

## 目录

- [鉴权](#鉴权)
  - [POST /management/console/auth/login](#post-managementconsoleauthlogin)
  - [POST /management/console/auth/logout](#post-managementconsoleauthlogout)
  - [GET /management/console/auth/me](#get-managementconsoleauthme)
- [系统角色变更](#系统角色变更)
  - [POST /management/console/users/{userId}/roles](#post-managementconsoleusersuseridroles)
- [身份组管理](#身份组管理)
  - [GET /management/console/identity-groups](#get-managementconsoleidentity-groups)
  - [POST /management/console/identity-groups](#post-managementconsoleidentity-groups)
  - [PATCH /management/console/identity-groups/{groupId}](#patch-managementconsoleidentity-groupsgroupid)
  - [DELETE /management/console/identity-groups/{groupId}](#delete-managementconsoleidentity-groupsgroupid)
  - [POST /management/console/users/{userId}/groups](#post-managementconsoleusersuseridgroups)
  - [DELETE /management/console/users/{userId}/groups/{groupId}](#delete-managementconsoleusersuseridgroupsgroupid)
- [配置热重载](#配置热重载)
  - [POST /management/console/reload](#post-managementconsolereload)
- [后台审计日志](#后台审计日志)
  - [GET /management/console/audit-logs](#get-managementconsoleaudit-logs)

---

## 鉴权

后台登录权限要求用户系统角色为 `admin`。后台不设额外级别——凡系统角色为 `admin` 的用户均可用其唯一后台账户进入。

### POST /management/console/auth/login

后台登录。

**请求**：

```json5
{
  "password": "系统生成的强密码"
}
```

**后端处理**：
1. 校验主站 Cookie：当前会话用户必须存在且系统角色为 `admin`，否则 401。
2. 校验后台账户存在且密码匹配（`console_admins`）。
3. 签发后台 JWT；在 `console_sessions` 登记会话；JWT 经响应头（如 `Authorization` 或自定义头）返回。

**响应**：`200`，响应头含 JWT：

```json5
{
  "jwt": "<jwt>", // 后台令牌，调用后台接口时放入 Authorization: Bearer
  "expiresIn": 3600,
  "role": "admin",
  "userId": "be081dbc-..."
}
```

**备注**：
- 密码错误或主站非 Admin：`401`，`error` 为 `Unauthorized`。
- 后台密码非用户本人设置，由后台生成/重置。

### POST /management/console/auth/logout

登出后台，使 JWT 失效。

**请求**：需携带主站 Cookie + 后台 JWT。

**后端处理**：吊销 `console_sessions` 中对应会话。

**响应**：`204`。

### GET /management/console/auth/me

获取当前后台登录信息。

**请求**：需携带主站 Cookie + 后台 JWT。

**响应**：`200`：

```json5
{
  "userId": "be081dbc-...",
  "username": "user_be08...",
  "role": "admin",
  "jwtJti": "s_01H..."
}
```

**备注**：Cookie 或 JWT 任一无效返回 `401`。

---

## 系统角色变更

### POST /management/console/users/{userId}/roles

变更用户系统角色（`user`/`helper`/`moderator`/`admin`）。

**权限**：绑定 `admin`。

**请求**：

```json5
{
  "role": "moderator",
  "reason": "任命版主" // 可选
}
```

**后端处理**：变更成功后清除该用户全部主站会话以保证即时生效，写主站审计。

**响应**：`200`：

```json5
{
  "userId": "be081dbc-...",
  "previousRole": "user",
  "role": "moderator",
  "changedAt": "2026-08-08T10:30:00Z"
}
```

**备注**：
- `role` 不在四者内返回 `400`，`error` 为 `InvalidRole`。
- 目标用户不存在返回 `404`，`error` 为 `UserNotFound`。
- 将用户设为/取消 `admin` 时应同步创建/删除其 `console_admins` 后台账户。

---

## 身份组管理

身份组元数据（`identity_groups`）的管理在此后台。权限节点列表由配置文件定义，**不包含在本组接口**；新增组后需在配置中补充其权限节点并[热重载](#配置热重载)。

### GET /management/console/identity-groups

身份组列表。

**权限**：双 `admin`。

**响应**：`200`，结构同 `GET /management/identity-groups`。

### POST /management/console/identity-groups

创建身份组。

**权限**：双 `admin`。

**请求**：

```json5
{
  "name": "groupA", // 唯一
  "displayName": "Group A" // 可选
}
```

**响应**：`201`，返回新组信息。

**备注**：名称已存在返回 `409`，`error` 为 `GroupNameTaken`。

### PATCH /management/console/identity-groups/{groupId}

改名身份组。

**权限**：双 `admin`。

**请求**：

```json5
{
  "name": "groupA2", // 改名后唯一
  "displayName": "Group A2"
}
```

**响应**：`200`。

**备注**：不存在返回 `404`，`error` 为 `GroupNotFound`。

### DELETE /management/console/identity-groups/{groupId}

删除身份组，并解除其下所有 `user_groups` 关联。

**权限**：双 `admin`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `GroupNotFound`。

### POST /management/console/users/{userId}/groups

为用户分配身份组。

**权限**：双 `admin`。

**请求**：

```json5
{
  "groupId": "g_01H..."
}
```

**响应**：`201`。

**备注**：
- 用户不存在返回 `404`，`error` 为 `UserNotFound`。
- 组不存在返回 `404`，`error` 为 `GroupNotFound`。
- 已分配返回 `409`，`error` 为 `GroupAlreadyAssigned`。

### DELETE /management/console/users/{userId}/groups/{groupId}

移除用户的身份组。

**权限**：双 `admin`。

**响应**：`204`。

**备注**：未分配返回 `404`，`error` 为 `GroupNotAssigned`。

---

## 配置热重载

### POST /management/console/reload

热重载全部配置文件（含身份组权限节点列表、权限节点表、限速、OIDC 等）。

**权限**：双 `admin`。

**请求**：无。

**响应**：`200`：

```json5
{
  "reloaded": ["permissions.toml", "rate-limit.toml", "OIDC.toml", "identity.toml"],
  "reloadedAt": "2026-08-08T10:30:00Z"
}
```

**备注**：任意文件解析失败返回 `400`，`error` 为 `ConfigParseError`，且不应用任何改动。

---

## 后台审计日志

### GET /management/console/audit-logs

后台审计日志查询（只读）。

**权限**：双 `admin`。

**请求参数**：`adminId`、`action`（前缀匹配）、`result`、`from`、`to`、`page` / `pageSize`。

**响应**：`200`，按 `createdAt` 倒序：

```json5
{
  "items": [
    {
      "id": "clog_01H...",
      "adminId": "be081dbc-...",
      "action": "console.role_change",
      "result": "success",
      "ip": "203.0.113.1",
      "userAgent": "Mozilla/5.0 ...",
      "payload": { "targetUserId": "...", "role": "moderator" },
      "createdAt": "2026-08-08T10:30:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 120
}
```

**备注**：action 取值参考下表：

| action | 说明 |
|--------|------|
| `console.login` / `console.logout` | 后台登录 / 登出 |
| `console.role_change` | 变更用户系统角色 |
| `console.group_create` / `console.group_rename` / `console.group_delete` | 身份组增删改 |
| `console.group_assign` / `console.group_remove` | 分配 / 移除身份组 |
| `console.reload` | 配置热重载 |

主站操作审计见 [主站审计日志](../index.md#审计日志)。