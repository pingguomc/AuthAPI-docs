# 端点：/management/bans

封禁管理端点。简化模型：一个用户同一时刻至多一条封禁记录，见 [bans 表](../SQL.md#封禁-bans)。

## 权限

本文件全部端点需权限节点 `management.bans`，默认最低系统角色 `Moderator`。

## 目录

- [POST /management/bans](#post-managementbans)
- [GET /management/bans](#get-managementbans)
- [GET /management/bans/{userId}](#get-managementbansuserid)
- [DELETE /management/bans/{userId}](#delete-managementbansuserid)

---

### POST /management/bans

创建封禁。

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

---

### GET /management/bans

封禁列表。

**请求参数**：`active`（可选）、`page` / `pageSize`。

**响应**：`200`，按 `createdAt` 倒序，结构同 POST 响应的条目标。

---

### GET /management/bans/{userId}

查询指定用户封禁状态。

**响应**：`200`，返回该用户的封禁对象；无封禁记录返回 `404`，`error` 为 `BanNotFound`。

---

### DELETE /management/bans/{userId}

解封（删除封禁记录）。

**请求**（可选）：

```json5
{
  "reason": "申诉通过"
}
```

**响应**：`204`。

**备注**：无该记录返回 `404`，`error` 为 `BanNotFound`。