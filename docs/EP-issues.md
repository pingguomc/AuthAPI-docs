# 端点：/issues

Issue（议题）系统**用户侧**端点，类 GitHub Issues。已登录用户均可创建与访问公开议题；私有议题仅**可见者**可访问（创建者 + 有 `issue.private_read` 权限节点者；**版主 Moderator 及以上默认拥有** `issue.private_read`）。

标签创建/删除、分配与议题开/关在 [/management](management/EP-issues.md)。

## 可见性

- **公开（public）**：所有登录用户可见、可评论。
- **私有（private）**：仅创建者与拥有 `issue.private_read` 节点者可见、可评论；他人访问返回 `404`（不暴露存在性）。`issue.private_read` 默认授予 `Moderator` 及以上系统角色。

## 目录

- [GET /issues](#get-issues)
- [GET /issues/mine](#get-issuesmine)
- [POST /issues](#post-issues)
- [GET /issues/{issueId}](#get-issuesissueid)
- [PATCH /issues/{issueId}](#patch-issuesissueid)
- [POST /issues/{issueId}/comments](#post-issuesissueidcomments)
- [GET /issues/{issueId}/comments](#get-issuesissueidcomments)
- [GET /issues/{issueId}/labels](#get-issuesissueidlabels)

---

### GET /issues

公开议题列表（不含私有）。支持按状态与标签筛选。

**请求**：查询参数：

| 参数                  | 类型     | 说明                                          |
|---------------------|--------|---------------------------------------------|
| `state`             | string | `open` / `closed`（默认 `open`）                |
| `label`             | string | 按标签名筛选                                      |
| `sort`              | string | `created`/`updated`/`comments`，默认 `created` |
| `q`                 | string | 标题/正文搜索                                     |
| `page` / `pageSize` | int    | 分页                                          |

**响应**：`200`：

```json5
{
  "total": 30,
  "page": 1,
  "pageSize": 20,
  "items": [
    {
      "id": "i_01H...",
      "title": "登录页在移动端错位",
      "state": "open",
      "createdAt": "2026-08-08T10:30:00Z",
      "creator": { "id": "...", "username": "user_be08..." },
      "labels": [ { "id": "...", "name": "bug", "color": "#dd3a0a" } ],
      "commentCount": 0
    }
  ]
}
```

### POST /issues

创建议题。

**请求**：

```json5
{
  "title": "登录页在移动端错位",
  "body": "Bug 描述...",
  "visibility": "public" // public / private
}
```

**响应**：`201`，返回新议题（结构同详情）。

**备注**：`title` 为空或过长返回 `400`。

### GET /issues/{issueId}

议题详情。

**响应**：`200`，返回：

```json5
{
  "id": "i_01H...",
  "title": "登录页在移动端错位",
  "body": "Bug 详情...",
  "visibility": "public",
  "state": "open",
  "closedReason": null, // completed / duplicated / not_planned
  "creator": { "id": "...", "username": "user_be08..." },
  "labels": [ { "id": "...", "name": "bug", "color": "#dd3a0a" } ],
  "createdAt": "2026-08-08T10:30:00Z",
  "updatedAt": "2026-08-08T10:30:00Z",
  "closedAt": null
}
```

**后端处理**：公开议题对登录用户可见；私有仅创建者或有 `issue.private_read` 者可见。

**备注**：无权限查看（含私有非可见者）返回 `404`，`error` 为 `IssueNotFound`。

### PATCH /issues/{issueId}

编辑议题（仅创建者可编辑 title / body / visibility）。

**请求**：

```json5
{
  "title": null, // 不传则不修改
  "body": "更新后的正文",
  "visibility": "public"
}
```

**响应**：`200`，返回更新后的议题。

**备注**：非创建者返回 `403`，`error` 为 `Forbidden`。

### POST /issues/{issueId}/comments

评论议题（可见者均可）。

**请求**：

```json5
{
  "content": "我来复现一下"
}
```

**响应**：`201`，返回评论对象。

### GET /issues/{issueId}/comments

议题评论列表。

**响应**：`200`：

```json5
{
  "total": 2,
  "items": [
    {
      "id": "ic_01H...",
      "user": { "id": "...", "username": "user_be08..." },
      "content": "我来复现一下",
      "createdAt": "2026-08-08T10:30:00Z"
    }
  ]
}
```

### GET /issues/{issueId}/labels

议题当前的标签列表。

**响应**：`200`：

```json5
{
  "labels": [ { "id": "...", "name": "bug", "color": "#dd3a0a" } ]
}
```

> 标签的分配（给议题打标签）由协管在管理侧完成，见 [管理侧](management/EP-issues.md)。

### GET /issues/mine

列出**当前用户创建**的议题（含其私有议题），供创建者查看自己发起的内容。

**请求**：查询参数同上（`state` / `label` / `sort` / `q` / `page` / `pageSize`，默认 `state=open`）。

**响应**：`200`，结构与 `GET /issues` 相同，但包含当前用户创建的所有私有议题。