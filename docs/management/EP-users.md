# 端点：/management/users

用户管理端点。从 `/management` 索引拆出，见 [management 索引](index.md)。

## 权限

本文件涉及的节点及默认最低系统角色：

| 权限节点               | 说明        | 默认最低系统角色    |
|--------------------|-----------|-------------|
| `management.users` | 用户列表 / 详情 | `Moderator` |
 
接口为**只读**（列表 / 详情）。用户字段的修改见对应端点：系统角色、身份组在后台 [/management/console](console/index.md)；前缀由用户自行修改（`EP-user.md`）。

## 目录

- [GET /management/users](#get-managementusers)
- [GET /management/users/{userId}](#get-managementusersuserid)

---

### GET /management/users

用户列表。

**权限**：`management.users`，默认最低系统角色 `Moderator`。

**请求**：无请求体，查询参数如下：

| 参数                 | 类型     | 说明                                         |
|--------------------|--------|--------------------------------------------|
| `page`             | int    | 页码，默认 1                                    |
| `pageSize`         | int    | 每页条数，默认 20，上限 100                          |
| `q`                | string | 邮箱精确匹配 **或** 用户名前缀匹配，不做全文模糊                |
| `role`             | string | 按系统角色筛选                                    |
| `status`           | string | `active` / `banned`（由 bans 即时计算）           |
| `registeredAfter`  | string | 注册时间下界，ISO 8601 UTC                        |
| `registeredBefore` | string | 注册时间上界，ISO 8601 UTC                        |
| `sortBy`           | string | 仅 `createdAt`、`lastLoginAt`，默认 `createdAt` |
| `order`            | string | `asc` / `desc`，默认 `desc`                   |

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

**后端处理**：读取用户基本信息、所属身份组、封禁状态（即时计算）。

**备注**：用户不存在返回 `404`，`error` 为 `UserNotFound`。