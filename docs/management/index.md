# 端点：/management

主站管理端点。本端点全部需要身份验证，采用 [Cookie HttpOnly 会话](../index.md#cookie-格式)。

## 权限

访问本端点下的任意接口要求会话用户拥有对应的 **权限节点**，否则返回 `403`，`error` 为 `Forbidden`。

权限判断采用并行双模型，实际节点 = 系统角色内建节点 ∪ 所属身份组节点，见 [权限体系](../SQL.md#权限体系说明)。

## 目录

管理端点按功能分文件：

- [用户管理（/management/users）](EP-users.md)
- [Yggdrasil 管理（/management/yggdrasil）](EP-yggdrasil.md)
- [身份组（/management/identity-groups）](#身份组)
- [前缀分配（/management/users/{userId}/prefix）](#前缀分配)
- [投票管理（/management/votes）](#投票管理)
- [Issue 管理（/management/issues）](#issue-管理)
- [封禁（/management/bans）](#封禁)
- [审计日志（/management/audit-logs）](#审计日志)

本文件内涉及的权限节点及默认最低系统角色（缺省为 `Moderator`）：

| 权限节点                         | 说明             | 默认最低系统角色                     |
|------------------------------|----------------|------------------------------|
| `management.identity_groups` | 身份组列表          | `Moderator`                  |
| `management.prefix.assign`   | 为给用户分配前缀       | `Moderator`（版主）              |
| `management.votes`           | 投票创建 / 管理      | `Moderator`（版主）              |
| `management.votes.data`      | 查看投票统计数据       | `Helper`（协管）                 |
| `management.issues`          | Issue 标签分配、开/关 | `Helper` ~ `Moderator`（分见下表） |
| `management.bans`            | 封禁管理           | `Moderator`                  |
| `management.audit_logs`      | 主站审计日志查询       | `Moderator`                  |

> 身份组与系统角色的**写入**（分配/变更）在后台 [/management/console](console/index.md)，本端点仅读出。

Issue 相关操作的角色映射：

| 操作           | 端点                               | 权限节点                   | 默认角色            |
|--------------|----------------------------------|------------------------|-----------------|
| 标签创建/删除      | 后台 console                       | `issues.labels.manage` | `Admin`（管理）     |
| 给议题分配标签      | `/management/issues/{id}/labels` | `issues.labels.assign` | `Helper`（协管）    |
| 议题打开/关闭（带原因） | `/management/issues/{id}/state`  | `management.issues`    | `Moderator`（版主） |
| 查看私有议题       | —                                | `issues.private_read`  | —（仅授予）          |

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

## 前缀分配

前缀由系统统一管理：预设清单在后台维护，本端点由版主从预设中分配给用户。见 [prefix_presets 表](../SQL.md#前缀预设-prefix_presets)。

### PUT /management/users/{userId}/prefix

为用户分配显示前缀（或清除）。

**权限**：`management.prefix.assign`，默认最低系统角色 `Moderator`（版主）。

**请求**：

```json5
{
  "prefix": "[VIP]" // 必须来自 prefix_presets；传 "" 表示清除
}
```

**后端处理**：校验前缀存在于 `prefix_presets`，更新 `users.prefix`。

**响应**：`204`。

**备注**：
- 用户不存在返回 `404`，`error` 为 `UserNotFound`。
- 前缀不在预设内返回 `400`，`error` 为 `InvalidPrefix`。

### GET /management/prefix-presets

获取可用前缀预设清单（供分配时选择；也可给前端展示）。

**权限**：`management.prefix.assign`，默认最低系统角色 `Moderator`。

**响应**：`200`：

```json5
{
  "prefixes": [ { "id": "p_01H...", "value": "[VIP]" } ]
}
```

---

## 投票管理

用户的投票交互见 [/votes](../EP-votes.md)。本端点管理投票的创建与数据查看。

### POST /management/votes

创建投票。

**权限**：`management.votes`，默认最低系统角色 `Moderator`（版主）。

**请求**：

```json5
{
  "title": "你觉得新界面怎么样？",
  "description": "可选",
  "optionType": "multiple", // single / multiple
  "endAt": "2026-08-15T00:00:00Z", // ISO 8601 UTC
  "options": [ { "content": "很喜欢" }, { "content": "一般" } ]
}
```

**响应**：`201`，返回投票对象（含选项 id）。

**备注**：`options` 至少 2 项；`optionType = single` 时用户单选。

### DELETE /management/votes/{voteId}

删除投票（含其选项与投票记录）。

**权限**：`management.votes`，默认最低系统角色 `Moderator`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `VoteNotFound`。

### GET /management/votes/{voteId}/data

查看投票**统计数据**（涉及粉丝最细粒度）。

**权限**：`management.votes.data`，默认最低系统角色 `Helper`（协管）。

**响应**：`200`：

```json5
{
  "id": "v_01H...",
  "title": "...",
  "total": 45,
  "options": [
    {
      "content": "很喜欢",
      "count": 30,
      "voters": [ { "userId": "...", "username": "user_be08..." } ] // 每选项投票人
    }
  ]
}
```

**备注**：`voters` 为各选项投票用户明细；不存在返回 `404`，`error` 为 `VoteNotFound`。

---

## Issue 管理

管理侧管理议题的标签分配与开/关。用户侧创建/浏览/评论见 [/issues](../EP-issues.md)。数据模型见 [SQL.md](../SQL.md#issue议题)。

### POST /management/issues/{issueId}/labels

给议题分配标签。

**权限**：`issues.labels.assign`，默认最低系统角色 `Helper`（协管）。

**请求**：

```json5
{
  "labelIds": ["l_01H...", "l_02H..."] // 覆盖式，传空数组表示清空
}
```

**响应**：`204`。

**备注**：议题不存在返回 `404`，`error` 为 `IssueNotFound`。

### PATCH /management/issues/{issueId}/state

打开 / 关闭议题（带关闭原因）。

**权限**：`management.issues`，默认最低系统角色 `Moderator`（版主）。

**请求**：

```json5
{
  "state": "closed", // open / closed
  "closedReason": "completed" // state=closed 时：completed / duplicated / not_planned
}
```

**响应**：`200`，返回议题当前状态。

**备注**：`state=open` 时 `closedReason` 置空并可重开；议题不存在返回 `404`，`error` 为 `IssueNotFound`。

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

| action                  | 对应接口                                              |
|-------------------------|---------------------------------------------------|
| `user.ban`              | POST /management/bans                             |
| `user.ban_delete`       | DELETE /management/bans/{userId}                  |
| `profile.rename`        | PATCH /management/yggdrasil/profiles/{profileId}  |
| `profile.delete`        | DELETE /management/yggdrasil/profiles/{profileId} |
| `texture.delete`        | DELETE /management/yggdrasil/textures/{hash}      |
| `user.role_change`      | 后台系统角色变更                                          |
| `identity.group_assign` | 后台身份组分配                                           |
| `user.notification`     | 后台发送站内通知                                          |
| `system.announcement`   | 后台发布全站公告                                          |

后台操作的审计见 [后台审计日志](console/index.md#get-managementconsoleaudit-logs)。