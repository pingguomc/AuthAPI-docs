# 端点： /management

本端点全部需要身份验证,采用 [Cookie HttpOnly 会话](./index.md#cookie-格式)。

访问本端点下的任意接口要求会话用户拥有每个节点指定的权限，否则返回 `403`。

## 目录

按 Helper > Moderator > Admin 的默认拥有节点排序。


## 获取用户列表

`GET /management/users`

需要权限 `management.users`，默认拥有者 `Moderator`。

用户列表。

**请求**：无请求体，查询参数如下：

|         参数         |   类型   | 说明                                           |
|:------------------:|:------:|----------------------------------------------|
|       `page`       |  int   | 页码,从 1 开始,默认 1                               |
|     `pageSize`     |  int   | 每页条数,默认 20,上限 100                            |
|        `q`         | string | 邮箱精确匹配 **或** 用户名前缀匹配,不做全文模糊                  |
|       `role`       | string | 按角色筛选                                        |
|      `status`      | string | 按状态筛选                                        |
| `registeredAfter`  | string | 注册时间下界,ISO 8601 UTC                          |
| `registeredBefore` | string | 注册时间上界,ISO 8601 UTC                          |
|      `sortBy`      | string | 仅接受 `createdAt`、`lastLoginAt`,默认 `createdAt` |
|      `order`       | string | `asc` / `desc`,默认 `desc`                     |

`sortBy` 取值不在白名单内返回 `400`。

**响应**: 成功返回HTTP状态码 `200`，响应体如下：

```json5
{
  "total": 1234,//符合条件的总数
  "page": 1,
  "pageSize": 20,
  "users": [
    {
      "id": "...",
      "username": "显示的用户名",
      "email": "user@example.com",
      "role": "user",
      "lastLoginAt": "2026-08-08T10:30:00Z", //从未登录为 null
      "createdAt": "2026-08-08T10:30:00Z"
    },
    {
      // ...
    }
  ]
}
```

**备注** :
* 列表其他信息，在详情接口提供。

## 获取用户信息

`GET /admin/users/{userId}`

需要权限 `management.users`，默认拥有者 `Moderator`。

用户详情。

**请求**：无请求体。

**响应**: 成功返回HTTP状态码 `200`:

```json5
{
  "id": "u_01H...",
  "displayName": "显示的用户名",
  "email": "user@example.com",
  "role": "user",
  "status": "active",
  "bannedUntil": null,
  "lastLoginAt": "2026-08-08T10:30:00Z",
  "lastLoginIp": "203.0.113.1",
  "registerIp": "203.0.113.1",
  "createdAt": "2026-08-08T10:30:00Z",
  "oidcBindings": [  //参见 ./EP-user.md#端点useroidc
    {
      "providerId": "github",
      "boundAt": "2026-08-08T10:30:00Z"
    }
  ]
}
```

用户不存在返回 `404`,`error` 为 `UserNotFound`。




