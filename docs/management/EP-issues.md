# 端点：/management/issues

Issue 管理端点。管理侧管理议题的`标签分配`与`开/关`。用户侧创建/浏览/评论见 [../EP-issues.md](../EP-issues.md)。数据模型见 [SQL.md](../SQL.md#issue议题)。

## 目录

- [POST /management/issues/{issueId}/labels](#post-managementissuesissueidlabels)
- [PATCH /management/issues/{issueId}/state](#patch-managementissuesissueidstate)

---

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

---

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