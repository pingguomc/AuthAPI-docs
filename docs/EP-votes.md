# 端点：/votes

投票系统**用户侧**端点。已登录用户均可访问。投票的创建与数据查看在 [/management](management/index.md#投票管理)。

## 权限

本端点默认对所有已登录用户开放（需要主站 Cookie 身份验证）。投票结果不公开——普通用户只能投票，不能查看他人选择或统计（统计查询需节点 `management.votes.data`，见管理侧）。

## 目录

- [GET /votes](#get-votes)
- [GET /votes/{voteId}](#get-votesvoteid)
- [POST /votes/{voteId}/answer](#post-votesvoteidanswer)
- [GET /votes/{voteId}/me](#get-votesvoteidme)

---

### GET /votes

列出当前进行中（未过期）的投票题目。

**请求**：查询参数 `page` / `pageSize`（可选，默认 1 / 20）。

**响应**：`200`：

```json5
{
  "total": 3,
  "page": 1,
  "pageSize": 20,
  "items": [
    {
      "id": "v_01H...",
      "title": "你觉得新界面怎么样？",
      "description": null,
      "optionType": "single", // single / multiple
      "endAt": "2026-08-15T00:00:00Z"
    }
  ]
}
```

**备注**：
- 仅返回未过期投票，不含统计与选项投票数。
- 已登录用户已投过的题目可附带 `answered: true`（可选）。

### GET /votes/{voteId}

获取单个投票题目与其选项。

**请求**：无请求体。

**响应**：`200`：

```json5
{
  "id": "v_01H...",
  "title": "你觉得新界面怎么样？",
  "description": "可选描述",
  "optionType": "multiple",
  "startAt": "2026-08-01T00:00:00Z",
  "endAt": "2026-08-15T00:00:00Z",
  "options": [
    { "id": "vo_01H...", "content": "很喜欢", "sortOrder": 0 },
    { "id": "vo_02H...", "content": "一般", "sortOrder": 1 }
  ]
}
```

**备注**：不返回每个选项的投票数（结果不公开）。

### POST /votes/{voteId}/answer

投票。

**请求**：

- `optionType = single`：`optionId` 提交一个选项
- `optionType = multiple`：`optionIds` 提交多个选项

```json5
// single
{ "optionId": "vo_01H..." }
```
```json5
// multiple
{ "optionIds": ["vo_01H...", "vo_02H..."] }
```

**后端处理**：校验投票未过期；校验选项属于该投票；写入 `vote_answers`；若已投过则覆盖（整份重选）。

**响应**：`204`，无响应体。

**备注**：
- 投票已结束返回 `400`，`error` 为 `VoteClosed`。
- 投票不存在返回 `404`，`error` 为 `VoteNotFound`。
- 选项不存在返回 `400`，`error` 为 `InvalidOption`。
- 用户重复提交视为覆盖（幂等）。

### GET /votes/{voteId}/me

查询当前用户在该投票中的选择。

**请求**：无请求体。

**响应**：`200`：

```json5
{
  "answered": true,
  "optionIds": ["vo_01H..."], // multiple 时为数组
  "answeredAt": "2026-08-08T10:30:00Z"
}
```

**备注**：未投票时 `answered: false`。