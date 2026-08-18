# 端点：/management

主站管理端点。本端点全部需要身份验证，采用 [Cookie HttpOnly 会话](../index.md#cookie-格式)。

## 权限

访问本端点下的任意接口要求会话用户拥有对应的**权限节点**，否则返回 `403`，`error` 为 `Forbidden`。

权限判断采用并行双模型，实际节点 = 系统角色内建节点 ∪ 所属身份组节点，见 [权限体系](../SQL.md#权限体系说明)。下表列出本端点涉及的节点及默认拥有它的最低系统角色（缺省为 `Moderator`）。

| 权限节点 | 说明 | 默认最低系统角色 |
|-----------|------|-----------|
| `management.users` | 用户列表 / 详情 | `Moderator` |
| `management.identity_groups` | 身份组列表 | `Moderator` |
| `management.yggdrasil.profiles` | 角色管理 | `Moderator` |
| `management.yggdrasil.textures` | 材质管理 | `Moderator` |
| `management.bans` | 封禁管理 | `Moderator` |
| `management.audit_logs` | 主站审计日志查询 | `Moderator` |

> 身份组与系统角色的**写入**（分配/变更）在后台 [/management/console](console/index.md)，本端点仅读出。

## 目录

- [用户管理](#用户管理)
  - [GET /management/users](#get-managementusers)
  - [GET /management/users/{userId}](#get-managementusersuserid)
- [身份组](#身份组)
  - [GET /management/identity-groups](#get-managementidentity-groups)
- [Yggdrasil 管理](#yggdrasil-管理)
  - [角色](#角色)
  - [材质](#材质)
- [封禁](#封禁)
- [审计日志](#审计日志)

---

## 用户管理

需要权限节点 `management.users`。接口为**只读**（列表 / 详情），用户字段的修改见对应端点（系统角色、身份组在后台；前缀见 `EP-user.md`）。

### GET /management/users

用户列表。

**权限**：`management.users`，默认最低系统角色 `Moderator`。

**请求**：无请求体，查询参数如下：

| 参数 | 类型 | 说明 |
|------|------|------|
| `page` | int | 页码，默认 1 |
| `pageSize` | int | 每页条数，默认 20，上限 100 |
| `q` | string | 邮箱精确匹配 **或** 用户名前缀匹配，不做全文模糊 |
| `role` | string | 按系统角色筛选 |
| `status` | string | `active` / `banned`（由 bans 即时计算） |
| `registeredAfter` | string | 注册时间下界，ISO 8601 UTC |
| `registeredBefore` | string | 注册时间上界，ISO 8601 UTC |
| `sortBy` | string | 仅 `createdAt`、`lastLoginAt`，默认 `createdAt` |
| `order` | string | `asc` / `desc`，默认 `desc` |

**响应**：

```json5
{
  "total": 1234,
  "page": 1,
  "pageSize": 20,
  "users": [
    {
      "id": "be081dbc-3de9-4138-9e13-3cbc5439dd4a",
      "username": "user_be08...",
      "email": "user@example.com",
      "role": "user", // 系统角色（非空）
      "prefix": "[前缀]", // 显示前缀（可空）
      "groups": ["groupA", "groupB"], // 所属身份组名（可为空）
      "status": "active",
      "bannedUntil": null,
      "lastLoginAt": "2026-08-08T10:30:00Z",
      "createdAt": "2026-08-08T10:30:00Z"
    }
  ]
}
```

**后端处理**：`role` / `status` 筛选均按即时值计算。

**备注**：
- 列表不返回 IP、OIDC 绑定等关联信息，仅在详情接口提供。
- 展示名一律用 `username`，显示格式 `[prefix]username[groups][role]`。

### GET /management/users/{userId}

用户详情。

**权限**：`management.users`，默认最低系统角色 `Moderator`。

**请求**：无请求体。

**响应**：

```json5
{
  "id": "be081dbc-3de9-4138-9e13-3cbc5439dd4a",
  "username": "user_be08...",
  "email": "user@example.com",
  "role": "user", // 系统角色（非空）
  "prefix": "[前缀]", // 可空
  "identityGroups": [ // 所属身份组详情（可空）
    { "id": "g_01H...", "name": "groupA", "displayName": "Group A" }
  ],
  "status": "active",
  "bannedUntil": null,
  "lastLoginAt": "2026-08-08T10:30:00Z",
  "lastLoginIp": "203.0.113.1",
  "registerIp": "203.0.113.1",
  "createdAt": "2026-08-08T10:30:00Z",
  "oidcBindings": [ // 参见 ../EP-user.md#端点useroidc
    { "providerId": "github", "boundAt": "2026-08-08T10:30:00Z" }
  ]
}
```

**备注**：用户不存在返回 `404`，`error` 为 `UserNotFound`。

---

## 身份组

需要权限节点 `management.identity_groups`。该端点仅**读出**身份组信息，组的创建/改名/删除与用户分配在后台 [/management/console](console/index.md)。

### GET /management/identity-groups

身份组列表。

**权限**：`management.identity_groups`，默认最低系统角色 `Moderator`。

**请求**：无请求体，查询参数 `page` / `pageSize`（可选）。

**响应**：

```json5
{
  "total": 5,
  "page": 1,
  "pageSize": 20,
  "groups": [
    { "id": "g_01H...", "name": "groupA", "displayName": "Group A" }
  ]
}
```

**备注**：身份组的权限节点列表由配置文件定义，本接口不返回权限节点。

---

## Yggdrasil 管理

管理 Yggdrasil 角色与材质资源，数据模型见 [SQL.md](../SQL.md) 的 `yggdrasil_*` 表。

> 去除了旧的“启动器会话管理”与“令牌管理”，令牌不落库，故不在此提供管理端点。

### 角色

#### GET /management/yggdrasil/profiles

角色列表。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**请求参数**：`userId`（可选）、`page` / `pageSize`。

**响应**：`200`：

```json5
{
  "total": 30,
  "page": 1,
  "pageSize": 20,
  "items": [
    {
      "id": "7b3f0c2f-...",
      "userId": "be081dbc-...",
      "name": "Steve",
      "model": "slim",
      "createdAt": "2026-08-08T10:30:00Z",
      "updatedAt": "2026-08-08T10:30:00Z"
    }
  ]
}
```

#### GET /management/yggdrasil/profiles/{profileId}

角色详情。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**响应**：

```json5
{
  "id": "7b3f0c2f-...",
  "userId": "be081dbc-...",
  "name": "Steve",
  "model": "slim",
  "textures": [
    { "textureType": "skin", "hash": "e051c27e...", "model": "slim" }
  ],
  "createdAt": "2026-08-08T10:30:00Z",
  "updatedAt": "2026-08-08T10:30:00Z"
}
```

**备注**：不存在返回 `404`，`error` 为 `ProfileNotFound`。

#### PATCH /management/yggdrasil/profiles/{profileId}

角色改名（名称全局唯一）。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**请求**：

```json5
{
  "name": "SteveNew"
}
```

**响应**：`200`，返回更新后的角色信息。

**备注**：
- 名称已存在返回 `409`，`error` 为 `ProfileNameTaken`。
- 改名后，绑定该角色的令牌（缓存）应标为**暂时失效**，令启动器刷新令牌以获取新名称。

#### DELETE /management/yggdrasil/profiles/{profileId}

删除角色。删除后其材质一并清理。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**请求**：无请求体。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `ProfileNotFound`。

### 材质

#### GET /management/yggdrasil/textures

材质列表（默认孤儿材质：引用的角色已删除或文件无记录）。

**权限**：`management.yggdrasil.textures`，默认最低系统角色 `Moderator`。

**请求参数**：`profileId`（可选）、`orphanOnly`（默认 `true`）、`page` / `pageSize`。

**响应**：`200`：

```json5
{
  "total": 8,
  "page": 1,
  "pageSize": 20,
  "items": [
    { "hash": "e051c27e...", "textureType": "skin", "profileId": null, "orphan": true }
  ]
}
```

#### DELETE /management/yggdrasil/textures/{hash}

删除材质文件（按 hash）。仅允许删除孤儿材质。

**权限**：`management.yggdrasil.textures`，默认最低系统角色 `Moderator`。

**后端处理**：若材质仍被角色引用则拒绝。

**响应**：
- 成功：`204`。
- 材质被引用：`409`，`error` 为 `TextureInUse`。
- 不存在：`404`，`error` 为 `TextureNotFound`。

---

## 封禁

简化模型：一个用户同一时刻至多一条封禁记录，见 [bans 表](../SQL.md#封禁-bans)。

### POST /management/bans

创建封禁。

**权限**：`management.bans`，默认最低系统角色 `Moderator`。

**请求**：

```json5
{
  "userId": "be081dbc-...",
  "bannedUntil": "2026-09-01T00:00:00Z", // null 表示永久封禁
  "reason": "违反社区规则" // 可选
}
```

**后端处理**：创建后吊销该用户全部令牌（强制下线），并写主站审计。

**响应**：`201`：

```json5
{
  "id": "ban_01H...",
  "userId": "be081dbc-...",
  "bannedUntil": "2026-09-01T00:00:00Z",
  "reason": "违反社区规则",
  "operatorId": "be081dbc-...",
  "createdAt": "2026-08-08T10:30:00Z"
}
```

**备注**：
- 目标用户不存在返回 `404`，`error` 为 `UserNotFound`。
- 该用户已有生效封禁返回 `409`，`error` 为 `BanAlreadyExists`。

### GET /management/bans

封禁列表。

**权限**：`management.bans`，默认最低系统角色 `Moderator`。

**请求参数**：`active`（可选）、`page` / `pageSize`。

**响应**：`200`，按 `createdAt` 倒序，结构同 POST 响应的条目标。

### GET /management/bans/{userId}

查询指定用户封禁状态。

**权限**：`management.bans`，默认最低系统角色 `Moderator`。

**响应**：`200`，返回该用户的封禁对象；无封禁记录返回 `404`，`error` 为 `BanNotFound`。

### DELETE /management/bans/{userId}

解封（删除封禁记录）。

**权限**：`management.bans`，默认最低系统角色 `Moderator`。

**请求**（可选）：

```json5
{
  "reason": "申诉通过"
}
```

**响应**：`204`。

**备注**：无该记录返回 `404`，`error` 为 `BanNotFound`。

---

## 审计日志

需要权限节点 `management.audit_logs`。

### GET /management/audit-logs

主站审计日志查询（只读）。

**权限**：`management.audit_logs`，默认最低系统角色 `Moderator`。

**请求参数**：`operatorId`、`targetUserId`、`action`（前缀匹配）、`result`（`success` / `denied`）、`from`、`to`、`page` / `pageSize`。

**响应**：`200`，按 `createdAt` 倒序：

```json5
{
  "items": [
    {
      "id": "log_01H...",
      "operatorId": "be081dbc-...",
      "action": "user.ban",
      "targetUserId": "be081dbc-...",
      "result": "success",
      "ip": "203.0.113.1",
      "payload": { "bannedUntil": "2026-09-01T00:00:00Z" },
      "createdAt": "2026-08-08T10:30:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 300
}
```

**备注**：
- `action` 取值见下表。
- 先业务后审计；只记鉴权层面拒绝；记录操作者 IP。

| action | 对应接口 |
|--------|----------|
| `user.ban` | POST /management/bans |
| `user.ban_delete` | DELETE /management/bans/{userId} |
| `profile.rename` | PATCH /management/yggdrasil/profiles/{profileId} |
| `profile.delete` | DELETE /management/yggdrasil/profiles/{profileId} |
| `texture.delete` | DELETE /management/yggdrasil/textures/{hash} |
| `user.role_change` | 后台系统角色变更 |
| `identity.group_assign` | 后台身份组分配 |

后台操作的审计见 [后台审计日志](console/index.md#get-managementconsoleaudit-logs)。