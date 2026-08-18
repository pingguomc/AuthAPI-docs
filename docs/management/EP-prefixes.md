# 端点：/management（前缀）

前缀指派于本文件。前缀为**多前缀模型**：预设由后台 SuperAdmin 维护（见 [console 前缀管理](console/index.md#前缀管理)），版主（Moderator）以上角色在此为用户**授予 / 收回**前缀；玩家持有多个前缀，并经 [PUT /user/me/prefix](../EP-user.md#put-usermeprefix-cookie身份验证) 选择佩戴或置空。

## 权限

前缀授予 / 收回 / 预设查询需权限节点 `management.prefix.assign`，默认最低系统角色 `Moderator`。

## 目录

- [GET /management/prefix-presets](#get-managementprefix-presets)
- [GET /management/users/{userId}/prefixes](#get-managementusersuseridprefixes)
- [POST /management/users/{userId}/prefixes](#post-managementusersuseridprefixes)
- [DELETE /management/users/{userId}/prefixes/{prefixId}](#delete-managementusersuseridprefixesprefixid)

---

### GET /management/prefix-presets

获取可用前缀预设清单（供授予时选择，也可给前端展示）。

**权限**：`management.prefix.assign`，默认最低系统角色 `Moderator`。

**响应**：`200`：

```json5
{
  "prefixes": [
    { "id": "p_01H...", "value": "[VIP]", "displayName": "VIP 用户", "backgroundColor": "#ffcc00" }
  ]
}
```

---

### GET /management/users/{userId}/prefixes

列出指定用户**已持有**的前缀。

**权限**：`management.prefix.assign`，默认最低系统角色 `Moderator`。

**响应**：`200`：

```json5
{
  "prefixes": [
    { "id": "p_01H...", "value": "[VIP]", "displayName": "VIP 用户", "backgroundColor": "#ffcc00" }
  ],
  "currentPrefixId": "p_01H..." // 该用户当前佩戴的前缀，可为 null
}
```

**备注**：用户不存在返回 `404`，`error` 为 `UserNotFound`。

---

### POST /management/users/{userId}/prefixes

为一个用户**授予**一个前缀（追加到该用户持有的前缀集合）。

**权限**：`management.prefix.assign`，默认最低系统角色 `Moderator`。

**请求**：

```json5
{
  "prefixId": "p_01H..." // 必须来自 prefix_presets
}
```

**响应**：`201`，返回该用户当前持有列表。

**备注**：
- 用户不存在返回 `404`，`error` 为 `UserNotFound`。
- 前缀预设不存在返回 `404`，`error` 为 `PrefixNotFound`。
- 该用户已持有此前缀返回 `409`，`error` 为 `PrefixAlreadyGranted`。

---

### DELETE /management/users/{userId}/prefixes/{prefixId}

收回用户持有的一个前缀。若其当前佩戴的正是该前缀，则佩戴同步置空。

**权限**：`management.prefix.assign`，默认最低系统角色 `Moderator`。

**响应**：`204`。

**备注**：
- 用户不存在返回 `404`，`error` 为 `UserNotFound`。
- 该用户未持有此前缀返回 `404`，`error` 为 `PrefixNotGranted`。