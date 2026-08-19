# 端点：/management

主站管理端点。本端点全部需要身份验证，采用 [Cookie HttpOnly 会话](../index.md#cookie-格式)。

## 权限

访问本端点下的任意接口要求会话用户拥有对应的 **权限节点**，否则返回 `403`，`error` 为 `Forbidden`。

权限判断采用并行双模型，实际节点 = 系统角色内建节点 ∪ 所属身份组节点，见 [权限体系](../SQL.md#权限体系说明)。

## 目录

管理端点按功能分文件：

- [用户管理（/management/users）](EP-users.md)
- [Yggdrasil 管理（/management/yggdrasil）](EP-yggdrasil.md)
- [前缀（/management/prefix-presets、授予/收回）](EP-prefixes.md)
- [投票管理（/management/votes）](EP-votes.md)
- [Issue 管理（/management/issues）](EP-issues.md)
- [封禁（/management/bans）](EP-bans.md)
- [审计日志（/management/audit-logs）](EP-audit-logs.md)
- [管理后台（/management/console）](console/index.md)

本文内涉及的权限节点及默认最低系统角色（缺省为 `Moderator`）：

| 权限节点                         | 说明             | 默认最低系统角色                     |
|------------------------------|----------------|------------------------------|
| `management.identity_groups` | 身份组列表          | `Moderator`                  |
| `management.prefix.assign`   | 前缀授予/收回        | `Moderator`（版主）              |
| `management.votes`           | 投票创建 / 管理      | `Moderator`（版主）              |
| `management.votes.data`      | 查看投票统计数据       | `Helper`（协管）                 |
| `management.issues`          | Issue 标签分配、开/关 | `Helper` ~ `Moderator`（分见下表） |
| `management.bans`            | 封禁管理           | `Moderator`                  |
| `management.audit_logs`      | 主站审计日志查询       | `Moderator`                  |

> 身份组与系统角色的**写入**（分配/变更）在后台 [/management/console](console/index.md)，本端点仅读出。前缀预设的维护同样在后台 console，本端点的前缀子模块仅负责**授予 / 收回**。

Issue 相关操作的角色映射：

| 操作           | 端点                              | 权限节点                   | 默认角色          |
| ------------- | -------------------------------- | ----------------------- | -------------- |
| 标签创建 / 删除    | 后台 console                        | `issues.labels.manage`  | `Admin`（管理）     |
| 给议题分配标签       | `/management/issues/{id}/labels`  | `issues.labels.assign`  | `Helper`（协管）    |
| 议题打开/关闭（带原因）  | `/management/issues/{id}/state`   | `management.issues`     | `Moderator`（版主） |
| 查看私有议题         | —                                | `issues.private_read`   | `Moderator`（版主） |

> `issues.private_read` 默认授予 `Moderator` 及以上。

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