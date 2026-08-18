# 端点：/management

主站管理端点。本端点全部需要身份验证，采用 [Cookie HttpOnly 会话](../index.md#cookie-格式)。

## 权限

访问本端点下的任意接口要求会话用户拥有对应的**权限节点**，否则返回 `403`，`error` 为 `Forbidden`。

权限判断采用并行双模型，实际节点 = 系统角色内建节点 ∪ 所属身份组节点，见 [权限体系](../SQL.md#权限体系说明)。

## 目录

管理端点按功能分文件：

- [用户管理（/management/users）](users.md)
- [Yggdrasil 管理（/management/yggdrasil）](yggdrasil.md)
- [身份组（/management/identity-groups）](#身份组)
- [封禁（/management/bans）](#封禁)
- [审计日志（/management/audit-logs）](#审计日志)

本文件内涉及的权限节点及默认最低系统角色（缺省为 `Moderator`）：

| 权限节点 | 说明 | 默认最低系统角色 |
|-----------|------|-----------|
| `management.identity_groups` | 身份组列表 | `Moderator` |
| `management.bans` | 封禁管理 | `Moderator` |
| `management.audit_logs` | 主站审计日志查询 | `Moderator` |

> 身份组与系统角色的**写入**（分配/变更）在后台 [/management/console](console/index.md)，本端点仅读出。

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

**后端处理**：创建后吊销该用户全部令牌（缓存，强制下线），并写主站审计。

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
| `user.notification` | 后台发送站内通知 |
| `system.announcement` | 后台发布全站公告 |

后台操作的审计见 [后台审计日志](console/index.md#get-managementconsoleaudit-logs)。