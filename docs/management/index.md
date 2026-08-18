# 端点：/management

主站管理端点。本端点全部需要身份验证，采用 [Cookie HttpOnly 会话](../index.md#cookie-格式)。

访问本端点下的任意接口要求会话用户拥有每个节点指定的**权限节点**，否则返回 `403`。

## 权限节点

权限节点为权限检查的最小单位，见 [系统角色、权限模型、用户身份](../index.md#系统角色权限模型用户身份)。下表列出本端点涉及的节点及默认拥有角色（缺省为 `Moderator`）。

| 权限节点 | 说明 | 默认拥有角色 |
|-----------|------|-----------|
| `management.users` | 用户列表 / 详情 | `Moderator` |
| `management.yggdrasil.launcher_sessions` | 启动器会话管理 | `Moderator` |
| `management.yggdrasil.profiles` | 角色管理 | `Moderator` |
| `management.yggdrasil.tokens` | 令牌管理 | `Moderator` |
| `management.yggdrasil.textures` | 材质管理 | `Moderator` |
| `management.bans` | 封禁管理 | `Moderator` |
| `management.audit_logs` | 主站审计日志查询 | `Moderator` |

若未拥有对应权限节点，返回 `403`，`error` 为 `Forbidden`。

## 目录

- [用户管理](#用户管理)
  - [GET /management/users](#get-managementusers)
  - [GET /management/users/{userId}](#get-managementusersuserid)
- [Yggdrasil 管理](#yggdrasil-管理)
  - [启动器会话](#启动器会话)
    - [GET /management/yggdrasil/launcher-sessions](#get-managementyggdrasillauncher-sessions)
    - [GET /management/yggdrasil/launcher-sessions/{launcherSessionId}](#get-managementyggdrasillauncher-sessionslaunchersessionid)
    - [DELETE /management/yggdrasil/launcher-sessions/{launcherSessionId}](#delete-managementyggdrasillauncher-sessionslaunchersessionid)
    - [POST /management/yggdrasil/launcher-sessions/{launcherSessionId}/reset-password](#post-managementyggdrasillauncher-sessionslaunchersessionidreset-password)
  - 涉及令牌的批量吊销见 [令牌管理](#令牌管理)
  - [角色](#角色)
    - [GET /management/yggdrasil/profiles](#get-managementyggdrasilprofiles)
    - [GET /management/yggdrasil/profiles/{profileId}](#get-managementyggdrasilprofilesprofileid)
    - [PATCH /management/yggdrasil/profiles/{profileId}](#patch-managementyggdrasilprofilesprofileid)
    - [DELETE /management/yggdrasil/profiles/{profileId}](#delete-managementyggdrasilprofilesprofileid)
  - [令牌](#令牌)
    - [GET /management/yggdrasil/tokens](#get-managementyggdrasiltokens)
    - [DELETE /management/yggdrasil/tokens/{accessToken}](#delete-managementyggdrasiltokensaccesstoken)
    - [POST /management/yggdrasil/tokens/revoke-all](#post-managementyggdrasiltokensrevoke-all)
  - [材质](#材质)
    - [GET /management/yggdrasil/textures](#get-managementyggdrasiltextures)
    - [DELETE /management/yggdrasil/textures/{hash}](#delete-managementyggdrasiltextureshash)
- [封禁](#封禁)
  - [POST /management/bans](#post-managementbans)
  - [GET /management/bans](#get-managementbans)
  - [GET /management/bans/{userId}](#get-managementbansuserid)
  - [DELETE /management/bans/{userId}](#delete-managementbansuserid)
- [审计日志](#审计日志)
  - [GET /management/audit-logs](#get-managementaudit-logs)

---

## 用户管理

需要权限节点 `management.users`。

### GET /management/users

用户列表。

**请求**：无请求体，查询参数如下：

| 参数                 | 类型     | 说明                                           |
|--------------------|--------|----------------------------------------------|
| `page`             | int    | 页码，从 1 开始，默认 1                               |
| `pageSize`         | int    | 每页条数，默认 20，上限 100                            |
| `q`                | string | 邮箱精确匹配 **或** 用户名前缀匹配，不做全文模糊                  |
| `role`             | string | 按系统角色筛选                                      |
| `status`           | string | 按状态筛选（`active`/`banned`，由 banned 即时计算）       |
| `registeredAfter`  | string | 注册时间下界，ISO 8601 UTC                          |
| `registeredBefore` | string | 注册时间上界，ISO 8601 UTC                          |
| `sortBy`           | string | 仅接受 `createdAt`、`lastLoginAt`，默认 `createdAt` |
| `order`            | string | `asc` / `desc`，默认 `desc`                     |

`sortBy` 取值不在白名单内返回 `400`。

**响应**：成功返回 HTTP 状态码 `200`，响应体如下：

```json5
{
  "total": 1234, // 符合条件的总数
  "page": 1,
  "pageSize": 20,
  "users": [
    {
      "id": "be081dbc-3de9-4138-9e13-3cbc5439dd4a",
      "username": "user_be08...",
      "email": "user@example.com",
      "role": "user",
      "status": "active", // banned / active（即时计算）
      "bannedUntil": null, // 封禁中则为时间，否则 null
      "lastLoginAt": "2026-08-08T10:30:00Z", // 从未登录为 null
      "createdAt": "2026-08-08T10:30:00Z"
    }
    // ...
  ]
}
```

**备注**：
- 列表不返回 IP 类字段与 OIDC 绑定、启动器会话等关联信息，仅在对应详情接口提供。
- `status = banned` 查询命中条件为：存在 `bans` 记录且 `ban_until IS NULL OR ban_until > now()`。

### GET /management/users/{userId}

用户详情。

**请求体**：无。

**响应**：成功返回 HTTP 状态码 `200`：

```json5
{
  "id": "be081dbc-3de9-4138-9e13-3cbc5439dd4a",
  "displayName": "显示的用户名",
  "email": "user@example.com",
  "role": "user",
  "status": "active",
  "bannedUntil": null,
  "lastLoginAt": "2026-08-08T10:30:00Z",
  "lastLoginIp": "203.0.113.1",
  "registerIp": "203.0.113.1",
  "permissions": [ // 该用户额外授予的权限节点（不含角色内建节点）
    "management.users"
  ],
  "createdAt": "2026-08-08T10:30:00Z",
  "launcherSessions": [ // 见 ../EP-server 关联文档；此处为摘要
    { "id": "7661e3a4-...", "email": "a1b2c3@example.com", "profileId": "..." }
  ],
  "oidcBindings": [ // 参见 ../EP-user.md#端点useroidc
    {
      "providerId": "github",
      "boundAt": "2026-08-08T10:30:00Z"
    }
  ]
}
```

用户不存在返回 `404`，`error` 为 `UserNotFound`。

---

## Yggdrasil 管理

Yggdrasil 相关资源（启动器会话、角色、令牌、材质）的管理，数据模型见 [SQL.md](../SQL.md) 的 `yggdrasil_*` 表，业务语义见 [yggdrasil/index.md](../yggdrasil/index.md)。

### 启动器会话

#### GET /management/yggdrasil/launcher-sessions

启动器会话列表。

**请求体**：无。查询参数：

| 参数         | 类型         | 说明                |
|------------|------------|-------------------|
| `userId`   | string（可选） | 按所属账号筛选           |
| `page`     | int        | 页码，默认 1           |
| `pageSize` | int        | 每页条数，默认 20，上限 100 |

**响应**：`200`：

```json5
{
  "total": 12,
  "page": 1,
  "pageSize": 20,
  "items": [
    {
      "id": "766e3a4e-...",
      "userId": "be081dbc-...",
      "email": "a1b2c3@example.com", // 启动器会话登录名
      "profileId": "7b3f0c2f-...",
      "profileName": "Steve",
      "createdAt": "2026-08-08T10:30:00Z"
    }
    // ...
  ]
}
```

#### GET /management/yggdrasil/launcher-sessions/{launcherSessionId}

单个启动器会话详情。

**响应**：`200`，结构同列表项，另含：

```json5
{
  "id": "a1b2c3d4-...",
  "userId": "be081798-...",
  "email": "a1b2c3@example.com",
  "profileId": "7b3f0c2f-...",
  "createdAt": "2026-08-08T10:30:00Z"
}
```

不存在返回 `404`，`error` 为 `LauncherSessionNotFound`。

#### DELETE /management/yggdrasil/launcher-sessions/{launcherSessionId}

删除启动器会话，并吊销其关联的**全部令牌**。

**响应**：`204`，无响应体。

**备注**：删除后该会话无法再登录。不存在返回 `404`，`error` 为 `LauncherSessionNotFound`。

#### POST /management/yggdrasil/launcher-sessions/{launcherSessionId}/reset-password

重置启动器会话认证凭据（生成新的随机密码）。

**响应**：`200`：

```json5
{
  "id": "a1b2c3d4-...",
  "password": "f81d4fae-..." // 新凭据，仅此一次返回
}
```

**备注**：重置后应同步吊销该会话的既有令牌以保证旧凭据失效。不存在返回 `404`。

### 角色

#### GET /management/yggdrasil/profiles

角色列表。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `userId` | string（可选） | 按所属账号筛选 |
| `page` | int | 页码，默认 1 |
| `pageSize` | int | 每页条数，默认 20，上限 100 |

**响应**：`200`：

```json5
{
  "total": 30,
  "page": 1,
  "pageSize": 20,
  "items": [
    {
      "id": "7b3f0c2f-...",
      "userId": "be081798-...",
      "name": "Steve",
      "model": "slim",
      "createdAt": "2026-08-08T10:30:00Z",
      "updatedAt": "2026-08-08T10:30:00Z"
    }
    // ...
  ]
}
```

#### GET /management/yggdrasil/profiles/{profileId}

角色详情。

**响应**：`200`，返回：

```json5
{
  "id": "7b3f0c2f-...",
  "userId": "be081798-...",
  "name": "Steve",
  "model": "slim",
  "textures": [ // 已上传材质（SKIN/CAPE）
    { "textureType": "skin", "hash": "e051c27e...", "model": "slim" }
  ],
  "createdAt": "2026-08-08T10:30:00Z",
  "updatedAt": "2026-08-08T10:30:00Z"
}
```

不存在返回 `404`，`error` 为 `ProfileNotFound`。

#### PATCH /management/yggdrasil/profiles/{profileId}

角色改名（名称全局唯一）。

**请求体**：

```json5
{
  "name": "SteveNew"
}
```

**响应**：`200`，返回新的角色信息。

**备注**：
- 名称已存在返回 `409`，`error` 为 `ProfileNameTaken`。
- 改名后，绑定该角色的令牌应被标记为**暂时失效**（`invalid_until`），令启动器刷新令牌以获取新名称。

#### DELETE /management/yggdrasil/profiles/{profileId}

删除角色。删除后其绑定的启动器会话、令牌、材质一并失效/清理。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `ProfileNotFound`。

### 令牌

> 令牌分布在不同启动器会话下，管理端可跨会话操作。

#### GET /management/yggdrasil/tokens

令牌列表。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `userId` | string（可选） | 按所属账号筛选 |
| `launcherSessionId` | string（可选） | 按启动器会话筛选 |
| `state` | string（可选） | `valid` / `invalid` / `temporarily` |
| `page` | int | 页码，默认 1 |
| `pageSize` | int | 每页条数，默认 20，上限 100 |

**响应**：`200`：

```json5
{
  "total": 50,
  "page": 1,
  "pageSize": 20,
  "items": [
    {
      "accessToken": "eyJ...", // accessToken 本身只作标识返回（不暴露到第三方）
      "clientToken": "f3d1...",
      "authRecordId": "a1b2c3d4-...",
      "profileId": "7b3f0c2f-...",
      "state": "valid",
      "createdAt": "2026-08-01T10:00:00Z",
      "expiresAt": "2026-08-16T10:00:00Z"
    }
    // ...
  ]
}
```

#### DELETE /management/yggdrasil/tokens/{accessToken}

吊销单个令牌。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `TokenNotFound`。

#### POST /management/yggdrasil/tokens/revoke-all

吊销指定账号的全部令牌（强制下线）。

**请求体**：

```json5
{
  "userId": "be081798-..." // 必填
}
```

**响应**：`200`：

```json5
{
  "userId": "be081798-...",
  "revokedCount": 12
}
```

**备注**：用户不存在返回 `404`，`error` 为 `UserNotFound`。

### 材质

#### GET /management/yggdrasil/textures

孤儿材质列表（引用的角色已删除、或文件无对应生产记录）。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `profileId` | string（可选） | 按角色筛选 |
| `orphanOnly` | bool | 仅孤儿材质，默认 `true` |
| `page` | int | 页码，默认 1 |
| `pageSize` | int | 每页条数，默认 20，上限 100 |

**响应**：`200`：

```json5
{
  "total": 8,
  "page": 1,
  "pageSize": 20,
  "items": [
    { "hash": "e051c27e...", "textureType": "skin", "profileId": null, "orphan": true }
    // ...
  ]
}
```

#### DELETE /management/yggdrasil/textures/{hash}

删除材质文件（按 hash）。仅允许删除孤儿材质，已被角色引用的材质返回 `409`，`error` 为 `TextureInUse`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `TextureNotFound`。

---

## 封禁

简化模型：一个用户同一时刻至多一条封禁记录，见 [bans 表](../SQL.md#封禁-bans)。

### POST /management/bans

创建封禁。

**请求体**：

```json5
{
  "userId": "be081798-...",
  "bannedUntil": "2026-09-01T00:00:00Z", // 解封时间，null 表示永久封禁
  "reason": "违反社区规则" // 可选
}
```

**响应**：`201`：

```json5
{
  "id": "ban_01H...",
  "userId": "be081798-...",
  "bannedUntil": "2026-09-01T00:00:00Z",
  "reason": "违反社区规则",
  "operatorId": "be081798-...",
  "createdAt": "2026-08-08T10:30:00Z"
}
```

**备注**：
- 目标用户不存在返回 `404`，`error` 为 `UserNotFound`。
- 该用户已有生效封禁记录返回 `409`，`error` 为 `BanAlreadyExists`。
- 创建成功即吊销该用户全部令牌（强制下线），并写主站审计。

### GET /management/bans

封禁列表。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `active` | bool | 仅返回生效中的记录 |
| `page` | int | 页码，默认 1 |
| `pageSize` | int | 每页条数，默认 20，上限 100 |

**响应**：`200`，按 `createdAt` 倒序：

```json5
{
  "total": 42,
  "page": 1,
  "pageSize": 20,
  "items": [
    { "id": "ban_01H...", "userId": "...", "bannedUntil": null, "reason": "...", "operatorId": "...", "createdAt": "..." }
    // ...
  ]
}
```

### GET /management/bans/{userId}

查询指定用户封禁状态。

**响应**：`200`，返回该记录的封禁对象（结构同 POST 响应）；若被查询用户无封禁记录，返回 `404`，`error` 为 `BanNotFound`。

### DELETE /management/bans/{userId}

解封（物理删除封禁记录）。

**请求体**（可选）：`{"reason": "申诉通过"}`

**响应**：`204`。

**备注**：不存在该封禁记录返回 `404`，`error` 为 `BanNotFound`；解封后用户恢复 `active`。

---

## 审计日志

需要权限节点 `management.audit_logs`。

### GET /management/audit-logs

主站审计日志查询（**只读**，见 [audit_logs](../SQL.md#主站审计日志-audit_logs)）。

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `operatorId` | string | 操作者 |
| `targetUserId` | string | 操作对象 |
| `action` | string | 动作，支持前缀匹配（如 `user.`） |
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
      "id": "log_01H...",
      "operatorId": "be081798-...",
      "action": "user.ban",
      "targetUserId": "be081798-...",
      "result": "success",
      "ip": "203.0.113.1",
      "payload": { "bannedUntil": "2026-09-01T00:00:00Z", "reason": "违反社区规则" },
      "createdAt": "2026-08-08T10:30:00Z"
    }
    // ...
  ],
  "page": 1,
  "pageSize": 20,
  "total": 300
}
```

**action 取值**（`资源.动作`）：

| action | 对应接口 |
|--------|----------|
| `user.ban` | [POST /management/bans](#post-managementbans) |
| `user.ban_delete` | [DELETE /management/bans/{userId}](#delete-managementbansuserid) |
| `launcher_session.delete` | 删除启动器会话 |
| `launcher_session.reset_password` | 重置启动器会话凭据 |
| `profile.rename` | 角色改名 |
| `profile.delete` | 角色删除 |
| `token.revoke` | 令牌吊销 |
| `texture.delete` | 材质删除 |

**写入规则**：
- 先做**业务**，后写审计；审计写入失败则回滚业务。
- 只记录鉴权层面的拒绝（权限不足等），普通 `400`/`404` 不写入。
- 审计记录操作者 IP。

后台操作的审计见 [后台审计日志](console/index.md#get-managementconsoleaudit-logs)。