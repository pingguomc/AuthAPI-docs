# 端点：/management/console

管理后台端点。仅系统角色为 `admin` 的用户可使用（经后台独立鉴权）。

## 鉴权机制

后台采用**独立鉴权**，与主站会话隔离：

- 一个系统角色为 `admin` 的用户，后台账户有且唯一（见 [console_admins](../../SQL.md#后台账户-console_admins)）。
- **登录**：校验主站 Cookie（确认已登录且为 `admin`）+ 输入后台密码，通过后签发后台 JWT。若该后台账户处于**待设密**状态（`must_change_password = true`，即新提升或已被重置），登录接口返回 `403`、`error` 为 `ConsolePasswordNotSet`，前端须引导其先走 `POST /management/console/auth/set-password`。
- **请求校验**：后台业务接口**同时校验**：
  1. 主站 Cookie——确认当前用户已登录主站且系统角色为 `admin`；
  2. 请求头 `Authorization: Bearer {JWT}`——校验后台会话有效（JWT 签名 + `jti` 在缓存中有效）。
- 两者都有效才放行；Cookie 或 JWT 任一无效返回 `401`。
- **后台级别**：`console_admins.role` 分 `admin`（普通管理）与 `super_admin`（超级管理）。用户被提升为 `admin` 时**创建后台账户并处于『待设密』状态**（首次进入后台须先设置强密码，非自动生成）；由后端终端 `/admin` 命令设置者为 `super_admin` 并随即生成强密码（见 [后端命令系统](../../backend.md)）。SuperAdmin 的后台密码**仅能由终端 `/admin` 控制**，后台接口不可改、不可重置。
- 后台操作一律写入[后台审计日志](../../SQL.md#后台审计日志-console_audit_logs)。

**缓存**：主站会话与后台 JWT 均**存于缓存，不落库**，见 [缓存](../../cache.md)。后台会话键 `console_session:{jti}`。

> **有效期说明**：后台 JWT **不自动续期**，`expiresIn` 固定（如 3600s）。JWT 先于主站 Cookie 过期时，需**重新登录后台**换取新 JWT（主站 Cookie 通常仍有效，可复用，无需重新登录主站；主站 Cookie 采用滑动过期自动续期，后台 JWT 不参与）。

## 命令概览

| 端点                           | 授权要求          |
|------------------------------|-----------------|
| auth/login、logout、me         | 已登录 admin（任意级别） |
| auth/set-password（首次设密）        | 主站 Cookie + 系统角色为 `admin`（待设密态） |
| auth/reset-password（重置普通 Admin）| `super_admin`   |
| users/{userId}/roles（变更系统角色） | 分级：`super_admin` 可改非 SuperAdmin；`admin` 可改非 Admin |
| identity-groups 增删改 / 分配移除   | `super_admin`   |
| notifications（发站内通知）         | `admin`（任意级别）   |
| announcements 发 / 查 / 删         | `super_admin`   |
| labels（标签建/删，系统角色 Admin）     | `admin`（任意级别）   |
| prefixes（前缀预设建/删）            | `super_admin`   |
| reload（配置热重载）                | `super_admin`   |
| audit-logs（后台审计）             | `admin`（任意级别）   |

## 目录

- [鉴权](#鉴权)
  - [POST /management/console/auth/login](#post-managementconsoleauthlogin)
  - [POST /management/console/auth/set-password](#post-managementconsoleauthset-password)
  - [POST /management/console/auth/reset-password](#post-managementconsoleauthreset-password)
  - [POST /management/console/auth/logout](#post-managementconsoleauthlogout)
  - [GET /management/console/auth/me](#get-managementconsoleauthme)
- [系统角色变更](#系统角色变更)
  - [POST /management/console/users/{userId}/roles](#post-managementconsoleusersuseridroles)
- [身份组管理](#身份组管理)
- [通知与公告](#通知与公告)
  - [POST /management/console/notifications](#post-managementconsolenotifications)
  - [POST /management/console/announcements](#post-managementconsoleannouncements)
  - [GET /management/console/announcements](#get-managementconsoleannouncements)
  - [DELETE /management/console/announcements/{id}](#delete-managementconsoleannouncementsid)
- [标签管理](#标签管理)
- [前缀管理](#前缀管理)
- [配置热重载](#配置热重载)
  - [POST /management/console/reload](#post-managementconsolereload)
- [后台审计日志](#后台审计日志)
  - [GET /management/console/audit-logs](#get-managementconsoleaudit-logs)

---

## 鉴权

### POST /management/console/auth/login

后台登录。

**请求**（凭据）：主站 Cookie + 后台密码。

```json5
{
  "password": "系统生成的强密码"
}
```

**后端处理**：
1. 校验主站 Cookie：当前会话用户必须存在且系统角色为 `admin`，否则 401。
2. 校验后台账户存在且密码匹配（`console_admins`）。
3. 签发后台 JWT；JWT 的 `jti` 写入缓存 `console_session:{jti}`，TTL = `expiresIn`。

**响应**：`200`，响应体含 JWT（调用方需保存并在请求头携带）：

```json5
{
  "jwt": "<jwt>", // 后续放入 Authorization: Bearer
  "expiresIn": 3600,
  "role": "admin", // 后台级别 admin / super_admin
  "userId": "be081dbc-..."
}
```

**备注**：
- 密码错误或主站非 Admin：`401`，`error` 为 `Unauthorized`。
- 若后台账户处于**待设密**状态（`must_change_password = true`，新提升或刚被重置）：返回 `403`、`error` 为 `ConsolePasswordNotSet`，并引导调用者执行 `set-password`。

### POST /management/console/auth/set-password

**首次设置 / 重置后重设**后台密码（顶替“首次进入后台手动设密”步骤）。

**请求**：主站 Cookie（要求系统角色为 `admin`）+ 请求体新密码。

```json5
{
  "password": "强密码"
}
```

**后端处理**：
1. 校验主站 Cookie 且系统角色为 `admin`；
2. 校验该后台账户处于**待设密**状态（或允许随时重设由实现决定，此处用于首次/重置场景）；
3. **强密码校验**：长度、复杂度达标（如 ≥12、含大小写/数字/符号），且经 Bcrypt 哈希比对**不得与主站密码一致**；
4. 写入 `console_admins.password_hash` 并清除 `must_change_password` 标记。

**响应**：`204`（此后可用正常 `login` 登录）。

**备注**：
- 密码强度不足返回 `400`，`error` 为 `ConsolePasswordWeak`。
- 与主站密码一致返回 `400`，`error` 为 `SameAsMainPassword`。

### POST /management/console/auth/reset-password

SuperAdmin 重置某个**普通 Admin** 的后台密码（重置后该账户回到「待设密」状态，下次进入后台先设新密码）。

**权限**：后台级别 `super_admin`。

**请求**：主站 Cookie + `Authorization: Bearer {JWT}`。

```json5
{
  "targetUserId": "be081dbc-..." // 被重置的普通 Admin 用户 ID
}
```

**后端处理**：校验目标为普通 `admin`（非 `super_admin`），清除其 `password_hash` 并置 `must_change_password = true`，写后台审计。

**响应**：`204`。

**备注**：
- 目标不存在返回 `404`，`error` 为 `UserNotFound`。
- 目标为 SuperAdmin 返回 `403`，`error` 为 `Forbidden`（SuperAdmin 密码只能由终端 `/admin` 控制）。

### POST /management/console/auth/logout

登出后台，使 JWT 失效。

**请求**：主站 Cookie + `Authorization: Bearer {JWT}`。

**后端处理**：删除缓存 `console_session:{jti}`。

**响应**：`204`。

### GET /management/console/auth/me

获取当前后台登录信息。

**请求**：主站 Cookie + `Authorization: Bearer {JWT}`。

**响应**：`200`：

```json5
{
  "userId": "be081dbc-...",
  "username": "user_be08...",
  "role": "admin", // 主站系统角色
  "consoleRole": "super_admin", // 后台级别
}
```

**备注**：Cookie 或 JWT 任一无效返回 `401`。

---

## 系统角色变更

### POST /management/console/users/{userId}/roles

变更用户系统角色（`user`/`helper`/`moderator`/`admin`）。

**权限**（分级）：
- 后台级别 `super_admin`：可变更任何**非 SuperAdmin** 用户（`user`/`helper`/`moderator`/`admin`）的系统角色；
- 后台级别 `admin`：可变更任何**非 Admin** 用户（`user`/`helper`/`moderator`）的系统角色，但**不可**提升为 `admin`、也**不可**变更现有 `admin`/`super_admin`；
- 两者都不可变更 SuperAdmin（SuperAdmin 及其后台密码仅由终端 `/admin` 控制）。

**请求**：

```json5
{
  "role": "moderator",
  "reason": "任命版主" // 可选
}
```

**后端处理**：变更成功后清除该用户全部主站会话（缓存）以保证即时生效，写主站审计。

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
- 超出授权范围（如普通 Admin 变更 Admin/SuperAdmin，或任何后台尝试变更 SuperAdmin）返回 `403`，`error` 为 `Forbidden`。
- 将用户设为 `admin` 时，同步创建其 `console_admins` 后台账户并置于**待设密**状态（`must_change_password = true`，首次进入后台须经 `set-password` 设置强密码）；将用户从 `admin` 降级时，同步删除其后台账户。

---

## 身份组管理

身份组元数据（`identity_groups`）在此后台管理。权限节点列表由配置文件定义，不在本组接口内；新增组后需在配置中补充其权限节点并[热重载](#配置热重载)。

### 身份组写入（仅 super_admin）

#### POST /management/console/identity-groups

创建身份组。

**权限**：后台级别 `super_admin`。

**请求**：

```json5
{
  "name": "groupA", // 唯一
  "displayName": "Group A" // 可选
}
```

**响应**：`201`，返回新组信息。

**备注**：名称已存在返回 `409`，`error` 为 `GroupNameTaken`。

#### PATCH /management/console/identity-groups/{groupId}

改名身份组。

**权限**：后台级别 `super_admin`。

**请求**：

```json5
{
  "name": "groupA2",
  "displayName": "Group A2"
}
```

**响应**：`200`。

**备注**：不存在返回 `404`，`error` 为 `GroupNotFound`。

#### DELETE /management/console/identity-groups/{groupId}

删除身份组，并解除其下所有 `user_groups` 关联。

**权限**：后台级别 `super_admin`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `GroupNotFound`。

#### POST /management/console/users/{userId}/groups

为用户分配身份组。

**权限**：后台级别 `super_admin`。

**请求**：

```json5
{
  "groupId": "g_01H..."
}
```

**响应**：`201`。

**备注**：用户不存在 `UserNotFound`；组不存在 `GroupNotFound`；已分配 `GroupAlreadyAssigned`（均对应 `404`/`409`）。

#### DELETE /management/console/users/{userId}/groups/{groupId}

移除用户的身份组。

**权限**：后台级别 `super_admin`。

**响应**：`204`。

**备注**：未分配返回 `404`，`error` 为 `GroupNotAssigned`。

---

## 通知与公告

### POST /management/console/notifications

给**单个用户**发送站内通知。

**权限**：后台级别 `admin`（含 `super_admin`）。

**请求**：

```json5
{
  "targetUserId": "be081dbc-...",
  "title": "欢迎加入社区",
  "content": "这里是内容"
}
```

**响应**：`201`，返回通知对象：

```json5
{
  "id": "n_01H...",
  "targetUserId": "be081dbc-...",
  "title": "欢迎加入社区",
  "createdAt": "2026-08-08T10:30:00Z"
}
```

**备注**：目标用户不存在返回 `404`，`error` 为 `UserNotFound`。

### POST /management/console/announcements

发布**全站公告**（所有用户可见）。

**权限**：后台级别 `super_admin`。

**请求**：

```json5
{
  "title": "维护通知",
  "content": "将于今晚 00:00 停机维护。",
  "published": true // 可选，默认 true
}
```

**响应**：`201`，返回公告对象：

```json5
{
  "id": "a_01H...",
  "title": "维护通知",
  "published": true,
  "publishedAt": "2026-08-08T10:30:00Z"
}
```

### GET /management/console/announcements

公告列表（后台查询，含未发布项）。

**权限**：后台级别 `super_admin`。

**请求参数**：`page` / `pageSize`。

**响应**：`200`，按 `publishedAt` 倒序：

```json5
{
  "total": 2,
  "page": 1,
  "pageSize": 20,
  "items": [
    { "id": "a_01H...", "title": "维护通知", "published": true, "publishedAt": "2026-08-08T10:30:00Z" }
  ]
}
```

### DELETE /management/console/announcements/{id}

删除一条全站公告。删除后不再对用户展示。

**权限**：后台级别 `super_admin`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `AnnouncementNotFound`。

---
## 标签管理

Issue 标签的创建 / 删除由系统角色 `Admin` 完成；分配标签给议题及打开 / 关闭议题见 [/management](../EP-issues.md)。

### GET /management/console/labels

标签列表。

**权限**：后台级别 `admin`（含 `super_admin`）。

**响应**：`200`：

```json5
{
  "labels": [ { "id": "l_01H...", "name": "bug", "color": "#dd3a0a" } ]
}
```

### POST /management/console/labels

创建标签。

**权限**：系统角色 `Admin`。

**请求**：

```json5
{
  "name": "bug", // 唯一
  "color": "#dd3a0a" // 可选，默认 #000000
}
```

**响应**：`201`，返回标签对象。

**备注**：名称已存在返回 `409`，`error` 为 `LabelNameTaken`。

### DELETE /management/console/labels/{labelId}

删除标签（并解除其与所有议题的关联）。

**权限**：系统角色 `Admin`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `LabelNotFound`。

---

## 前缀管理

前缀预设由后台 SuperAdmin 维护，版主在 `/management` 为用户分配。

### GET /management/console/prefixes

前缀预设列表。

**权限**：后台级别 `super_admin`。

**响应**：`200`：

```json5
{
  "prefixes": [ { "id": "p_01H...", "value": "[VIP]" } ]
}
```

### POST /management/console/prefixes

创建前缀预设。

**权限**：后台级别 `super_admin`。

**请求**：

```json5
{
  "value": "[VIP]", // 唯一
  "displayName": "VIP 用户", // 可选，展示名
  "backgroundColor": "#ffcc00" // 可选，默认 #000000
}
```

**响应**：`201`，返回前缀对象。

**备注**：值已存在返回 `409`，`error` 为 `PrefixTaken`。

### DELETE /management/console/prefixes/{prefixId}

删除前缀预设（该预设下所有 `user_prefixes` 关联一并清除；佩戴该前缀的用户其 `users.prefix_id` 同步置空）。

**权限**：后台级别 `super_admin`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `PrefixNotFound`。

---

## 配置热重载

### POST /management/console/reload

热重载全部配置文件（含身份组权限节点、权限节点表、限速、OIDC 等）。

**权限**：后台级别 `super_admin`。

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

**权限**：后台级别 `admin`（含 `super_admin`）。

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

**备注**：`action` 取值参考下表：

| action                                                                   | 说明         |
|--------------------------------------------------------------------------|------------|
| `console.login` / `console.logout`                                       | 后台登录 / 登出  |
| `console.role_change`                                                    | 变更用户系统角色   |
| `console.group_create` / `console.group_rename` / `console.group_delete` | 身份组增删改     |
| `console.group_assign` / `console.group_remove`                          | 分配 / 移除身份组 |
| `console.notification_send`                                              | 发站内通知      |
| `console.announcement_publish`                                           | 发全站公告      |
| `console.reload`                                                         | 配置热重载      |

主站操作审计见 [主站审计日志](../EP-audit-logs.md)。