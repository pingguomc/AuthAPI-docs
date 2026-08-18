# 端点：/management/audit-logs

主站审计日志查询（只读）。需权限节点 `management.audit_logs`，默认最低系统角色 `Moderator`。

## GET /management/audit-logs

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

| action                  | 对应接口                                             |
|-------------------------|--------------------------------------------------|
| `user.ban`              | POST /management/bans                            |
| `user.ban_delete`       | DELETE /management/bans/{userId}                 |
| `profile.rename`        | PATCH /management/yggdrasil/profiles/{profileId} |
| `profile.delete`        | DELETE /management/yggdrasil/profiles/{profileId}|
| `texture.delete`        | DELETE /management/yggdrasil/textures/{hash}     |
| `user.role_change`      | 后台系统角色变更                                        |
| `identity.group_assign` | 后台身份组分配                                          |
| `user.notification`     | 后台发送站内通知                                          |
| `system.announcement`   | 后台发布全站公告                                          |

后台操作的审计见 [后台审计日志](console/index.md#get-managementconsoleaudit-logs)。