# 端点：/management/votes

投票管理端点。投票的创建与数据查看在此；用户侧投票交互见 [../EP-votes.md](../EP-votes.md)。

## 权限

| 权限节点 | 说明 | 默认最低系统角色 |
|------|------|----------|
| `management.votes` | 投票创建 / 删除 | `Moderator`（版主） |
| `management.votes.data` | 查看投票统计数据 | `Helper`（协管） |

## 目录

- [POST /management/votes](#post-managementvotes)
- [DELETE /management/votes/{voteId}](#delete-managementvotesvoteid)
- [GET /management/votes/{voteId}/data](#get-managementvotesvoteiddata)

---

### POST /management/votes

创建投票。

**权限**：`management.votes`，默认最低系统角色 `Moderator`（版主）。

**请求**：

```json5
{
  "title": "你觉得新界面怎么样？",
  "description": "可选",
  "optionType": "multiple", // single / multiple
  "maxSelections": 2, // 多选时最多可选选项数；0 或省略表示不限制
  "startAt": "2026-08-01T00:00:00Z", // 可选，开始时间（缺省即立即生效）
  "endAt": "2026-08-15T00:00:00Z", // ISO 8601 UTC
  "options": [ { "content": "很喜欢" }, { "content": "一般" } ]
}
```

**响应**：`201`，返回投票对象（含选项 id、`maxSelections`、`startAt`）。

**备注**：`options` 至少 2 项；`optionType = single` 时用户单选（`maxSelections` 固定为 1）。**`startAt` 需被遵守**——未到开始时间的投票不会出现在用户侧列表，也不可被投票。

---

### DELETE /management/votes/{voteId}

删除投票（含其选项与投票记录）。

**权限**：`management.votes`，默认最低系统角色 `Moderator`。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `VoteNotFound`。

---

### GET /management/votes/{voteId}/data

查看投票**统计数据**（涉及最细粒度投票人明细）。

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