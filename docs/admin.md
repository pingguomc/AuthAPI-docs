# 端点：/admin

管理端点。本端点全部需要身份验证,采用 [Cookie HttpOnly 会话](./index.md#cookie-格式)。

访问本端点下的任意接口要求会话用户 `role` 为 `admin`,否则返回 `403`。

## 目录

- [数据模型](#数据模型)
- [GET /admin/users](#get-adminusers)
- [GET /admin/users/{userId}](#get-adminusersuserid)
- [端点:/admin/bans](#端点adminbans)
    - [POST /admin/bans/{userId}](#post-adminbansuserid)
    - [GET /admin/bans](#get-adminbans)
    - [GET /admin/bans/{userId}](#get-adminbansuserid)
    - [DELETE /admin/bans/{userId}](#delete-adminbansuserid)
- [POST /admin/session-revocations/{userId}](#post-adminsession-revocationsuserid)
- [POST /admin/role-grants/{userId}](#post-adminrole-grantsuserid-下个版本废弃)
- [GET /admin/audit-logs](#get-adminaudit-logs)

## 数据模型

### role

用户角色,与状态正交。取值:

|      值      | 说明          |
|:-----------:|-------------|
|   `admin`   | 管理员,可访问管理后台 |
| `moderator` | 协管/版主       |
|   `user`    | 普通用户        |

### status

用户状态,与角色正交。取值:

|    值     | 说明  |
|:--------:|:---:|
| `active` | 正常  |
| `banned` | 封禁中 |

`bannedUntil` 为可空字段,`null` 表示永久封禁。

**status 不可直接写入**。 封禁记录(ban)是事实来源,`users.status` 与 `bannedUntil` 仅为其投影,由服务层在写入 ban 时同步维护。本文档不提供任何直接修改 `status` 的接口。

临时封禁到期采用惰性回写:仅在 **会话鉴权/登录** 与 [GET /admin/users](#get-adminusers) 两处检测到 `bannedUntil` 已过期时,就地将 `status` 改回 `active`。列表接口不回写,其正确性由查询条件保证。

所有返回 `status` 的接口均返回 **计算过期后的有效值** ,前端不需要自行比较 `bannedUntil` 与当前时间。

## GET /admin/users

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
      "id": "u_01H...",
      "displayName": "显示的用户名",
      "email": "user@example.com",
      "role": "user",
      "status": "banned",
      "bannedUntil": "2026-09-01T00:00:00Z", //null 表示永久封禁;status 非 banned 时为 null
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
* 列表不返回 IP 类字段与 OIDC 绑定信息，二者仅在详情接口提供。
* `status = banned` 的筛选条件需实现为 `status = 'banned' AND (bannedUntil IS NULL OR bannedUntil > now())`。
* 本接口不触发过期回写。

## GET /admin/users/{userId}

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
  "oidcBindings": [  //参见 ./user.md#端点useroidc
    {
      "providerId": "github",
      "boundAt": "2026-08-08T10:30:00Z"
    }
  ]
}
```

用户不存在返回 `404`,`error` 为 `UserNotFound`。

**备注**: 本接口会触发临时封禁的过期回写。

## 端点：/admin/bans

封禁记录。  
同一用户同时只允许存在一条生效中的封禁记录。以 `{userId}` 标识封禁对象。删除封禁记录即同步将用户 `status` 改回 `active`、`bannedUntil` 置 `null`，并写入审计日志。

### POST /admin/bans/{userId}

创建封禁。

**请求**:

```json5
{
  "bannedUntil": "2026-09-01T00:00:00Z", // 解封时间,null 表示永久封禁
  "reason": "违反社区规则" // 可选
}
```

**响应**: 成功返回HTTP状态码 `201`:

```json5
{
  "id": "ban_01H...",
  "userId": "u_01H...",
  "bannedUntil": "2026-09-01T00:00:00Z",
  "reason": "违反社区规则",
  "operatorId": "u_01H...",
  "createdAt": "2026-08-08T10:30:00Z"
}
```

**备注**:
* 同一用户同时只允许存在一条生效中的封禁记录。已存在时返回 `409`,`error` 为 `BanAlreadyExists`。修改封禁时长需先删除原记录再重新创建。
* 创建成功会同步更新 `users.status` 与 `users.bannedUntil`,并**清除该用户的全部会话**,该次会话清除不额外产生 `user.session_revoke` 审计记录。
* 目标用户不存在返回 `404`,`error` 为 `UserNotFound`。

### GET /admin/bans

查询封禁记录列表。

**请求参数**(Query):

|     参数     |  类型  | 说明           |
|:----------:|:----:|--------------|
|  `active`  | bool | 仅返回/排除生效中的记录 |
|   `page`   | int  | 页码,默认 1      |
| `pageSize` | int  | 每页条数,默认 20   |

**响应**:成功返回HTTP状态码 `200`,`banlists` 为封禁记录对象,结构同 [POST /admin/bans/{userId}](#post-adminbansuserid) 响应,固定按 `createdAt` 倒序:

```json5
{
  "banlists": [
    {
      "id": "ban_01H...",
      "userId": "u_01H...",
      "bannedUntil": "2026-09-01T00:00:00Z",
      "reason": "违反社区规则",
      "operatorId": "u_01H...",
      "createdAt": "2026-08-08T10:30:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 42
}
```

### GET /admin/bans/{userId}

该用户是否被封禁。

**请求**：无请求体。

**响应**：同 [POST /admin/bans/{userId}](#post-adminbansuserid) 的响应。

### DELETE /admin/bans/{userId}

物理删除封禁记录(解封)。记录被删除后不可恢复,操作会写入审计日志。

**请求**:

```json5
{
  "reason": "申诉通过"  //可选,记录删除原因
}
```

**响应**:成功返回HTTP状态码 `204`,无响应体。

**备注**:
* 不存在该用户的封禁记录时返回 `404`,`error` 为 `BanNotFound`。
* 删除成功会同步将 `users.status` 改回 `active`、`bannedUntil` 置 `null`。
* 删除需先通过鉴权;若鉴权层面被拒绝(权限不足、角色不匹配),会写入 `denied` 审计记录。
* 删除不清除会话。

## POST /admin/session-revocations/{userId}

强制下线,清除指定用户的全部会话。

**请求**:

```json5
{
  "reason": "疑似账号泄露"  //可选
}
```

**响应**: 成功返回HTTP状态码 `200`:

```json5
{
  "userId": "u_01H...",
  "revokedCount": 3,
  "createdAt": "2026-08-08T10:30:00Z"
}
```

**备注**: 会话清除同时也是封禁、角色变更的副作用,以及用户自行 [修改密码](./user.md#post-userchange-password-cookie身份验证) 的副作用。**仅通过本接口显式发起的清除** 才产生 `user.session_revoke` 审计记录,作为副作用触发的不重复记录。

## POST /admin/role-grants/{userId} (下个版本废弃)

变更用户角色。

**请求**:

```json5
{
  "role": "moderator",
  "reason": "任命版主"  //可选
}
```

**响应**:成功返回HTTP状态码 `201`:

```json5
{
  "id": "rg_01H...",
  "userId": "u_01H...",
  "previousRole": "user",
  "role": "moderator",
  "operatorId": "u_01H...",
  "createdAt": "2026-08-08T10:30:00Z"
}
```

**备注**:
* `role` 取值不在 `admin` / `moderator` / `user` 内返回 `400`。
* 变更成功会 **清除该用户的全部会话** ,以保证降权即时生效。该次会话清除不额外产生 `user.session_revoke` 审计记录。
* 当前 **不限制** 管理员变更自身或其他管理员的角色。请注意由此产生的锁死风险(例如唯一管理员将自己降级后无人可进入后台),此情况需手动修改数据库恢复。

## GET /admin/audit-logs

审计日志查询。

审计日志 **只读**,不提供任何创建、修改、删除接口。

**请求**：无请求体，查询参数如下：

|       参数       |   类型   | 说明                   |
|:--------------:|:------:|----------------------|
|  `operatorId`  | string | 操作者                  |
| `targetUserId` | string | 操作对象                 |
|    `action`    | string | 动作,支持前缀匹配(如 `user.`) |
|    `result`    | string | `success` / `denied` |
|     `from`     | string | 时间下界,ISO 8601 UTC    |
|      `to`      | string | 时间上界,ISO 8601 UTC    |
|     `page`     |  int   | 页码,默认 1              |
|   `pageSize`   |  int   | 每页条数,默认 20           |

固定按 `createdAt` 倒序,不开放排序参数。

**响应**: 成功返回HTTP状态码 `200`:

```json5
{
  "items": [
    {
      "id": "log_01H...",
      "operatorId": "u_01H...",
      "action": "user.ban",
      "targetUserId": "u_01H...",
      "result": "success",
      "ip": "203.0.113.1",
      //操作者 IP
      "payload": {
        //动作相关参数,结构随 action 而异
        "bannedUntil": "2026-09-01T00:00:00Z",
        "reason": "违反社区规则"
      },
      "createdAt": "2026-08-08T10:30:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 42
}
```

### action 取值

|           值           | 对应接口                                                                             |
|:---------------------:|----------------------------------------------------------------------------------|
|      `user.ban`       | [POST /admin/bans/{userId}](#post-adminbansuserid)                               |
|   `user.ban_delete`   | [DELETE /admin/bans/{userId}](#delete-adminbansuserid)                           |
| `user.session_revoke` | [POST /admin/session-revocations/{userId}](#post-adminsession-revocationsuserid) |
|  `user.role_change`   | [POST /admin/role-grants/{userId}](#post-adminrole-grantsuserid-下个版本废弃)          |

命名采用 `资源.动作` 形式。

### 写入规则

* **先做业务，后写审计。** 审计写入失败则整个操作失败，回滚业务逻辑。
* **只记录鉴权层面的拒绝**，即权限不足、角色不匹配等。参数校验失败、目标不存在等普通 `400` / `404` 不写入审计。
* 审计日志记录操作者 IP。