# 持久数据库表结构及模型参考

表名用小写+复数。时间一律采用 ISO 8601 UTC 字符串（`VARCHAR(32)`）存储，与 API 约定一致。

## 权限体系说明

采用**并行双模型**：

- **系统角色（role）**：`user` / `helper` / `moderator` / `admin`，权限节点表相对固定（内建），用户必有一个且只能有一个。
- **身份组（identity group）**：相当于自定义角色，一名用户可属于**多个**身份组（也可不属任何组）。身份组的元数据（id、名称）存于 `identity_groups` 表，其**权限节点列表不落库**，由配置文件逐组定义。

实际权限（判断某端点/节点是否放行）为：**系统角色内建节点 ∪ 该用户所属全部身份组节点**（并集），任一来源拥有即放行。

## 用户 (users)

| 列名            | 约束                                                   | 描述/备注                                  |
|---------------|------------------------------------------------------|----------------------------------------|
| id            | 主键                                                   | 随机的 UUID                               |
| email         | 唯一，可为空                                               | 主站登录邮箱                                 |
| password_hash | 可为空                                                  | Bcrypt 哈希                              |
| username      | 唯一，不为空，默认 `user_它的id`                                | 展示用用户名（对外直接展示它）                        |
| display_name  | 可为空                  | 仅保留字段（兼容），不再用于展示；展示一律用 `username`                |
| role          | 枚举 `user`/`helper`/`moderator`/`admin`，不为空，默认 `user` | **系统角色（非空）**                           |
| prefix_id     | 可为空，外键 `prefix_presets(id)` | 当前**佩戴**的前缀（须为该用户已持有的 `user_prefixes` 之一；玩家可自行选择佩戴或置空） |
| last_login_at | 可为空                  | 最后登录时间                                 |
| created_at    | 不为空                  | 注册时间                                   |
| updated_at    | 不为空                  | 更新时间                                   |

> 前端展示格式：`[前缀]username[身份组][系统角色]`（前缀为该用户当前佩戴的那个；身份组多组并集展示）。

用户是否被封禁不冗余存储；封禁状态一律由 `bans` 表即时计算得出。

## OIDC 记录 (oidc_records)

| 列名          | 约束                 | 描述/备注                                  |
|-------------|--------------------|----------------------------------------|
| id          | 自增 INT 主键          |                                        |
| user_id     | 不为空，外键 `users(id)` |                                        |
| provider_id | 不为空                |                                        |
| issuer      | 不为空                |                                        |
| subject     | 不为空                |                                        |
| username    | 可为空                | 通过该 OIDC 直接注册时采用此用户名；非 OIDC 首次注册则此项不使用 |
| created_at  | 不为空                |                                        |
| updated_at  | 不为空                |                                        |

> **唯一性**：`(issuer, subject)` 应建立 `UNIQUE` 唯一约束，并在绑定/登录时使用 upsert，防止同一外部账号被并发绑定到多个本地用户（账号接管/绑定竞态）。

## 身份组 (identity_groups)

自定义角色（身份组）的元数据。权限节点列表**不落库**，由配置文件按组定义。

| 列名           | 约束              | 描述/备注   |
|--------------|-----------------|---------|
| id           | 主键              | 组 ID    |
| name         | NOT NULL，UNIQUE | 组名称（唯一） |
| display_name | 可为空             | 展示名     |
| created_at   | 不为空             |         |
| updated_at   | 不为空             |         |

## 用户-身份组关联 (user_groups)

一名用户 ↔ 多个身份组（多对多）。

| 列名         | 约束                          | 描述/备注            |
|------------|-----------------------------|------------------|
| user_id    | 主键，外键 `users(id)`           | 用户               |
| group_id   | 主键，外键 `identity_groups(id)` | 身份组              |
| granted_by | 可为空，外键 `users(id)`          | 分配操作者（后台账户对应的用户） |
| granted_at | 不为空                         | 分配时间             |

> `(user_id, group_id)` 复合主键；同一用户同一组仅一条。

## 前缀预设与用户持有 (prefix_presets / user_prefixes)

前缀为**多前缀模型**：预设清单由后台 SuperAdmin 维护；版主以上角色为用户**授予 / 收回**前缀；玩家持有多个前缀，可佩戴其中一个或置空。预设删除时，其下所有持有关联与佩戴一并清除。

### 前缀预设 (prefix_presets)

| 列名               | 约束                 | 描述/备注              |
|------------------|--------------------|--------------------|
| id               | 主键                 | 随机 UUID            |
| value            | NOT NULL，UNIQUE    | 前缀字符串，如 `[VIP]`    |
| display_name     | 可为空                | 前缀展示名              |
| background_color | 可为空，默认 `#000000`   | 前缀背景色（十六进制）        |
| created_by       | 可为空，外键 `users(id)` | 创建者（后台 SuperAdmin） |
| created_at       | 不为空                |                    |

### 用户持有-前缀关联 (user_prefixes)

一名用户 ↔ 多个前缀（多对多），可持有 0..N 个。

| 列名         | 约束                         | 描述/备注 |
|------------|----------------------------|-------|
| user_id    | 主键，外键 `users(id)`          | 用户    |
| prefix_id  | 主键，外键 `prefix_presets(id)` | 持有的前缀 |
| granted_by | 可为空，外键 `users(id)`         | 授予操作者 |
| granted_at | 不为空                        | 授予时间  |

> `(user_id, prefix_id)` 复合主键；同一用户同一前缀仅一条。当前佩戴记录在 `users.prefix_id`，必须为该用户持有的 `user_prefixes` 之一。

### 投票 (votes)

| 列名             | 约束                                          | 描述/备注                  |
|----------------|---------------------------------------------|------------------------|
| id             | 主键                                          | 随机 UUID                |
| title          | 不为空                                         | 投票标题                   |
| description    | 可为空                                         | 描述                     |
| option_type    | NOT NULL，枚举 `single`/`multiple`，默认 `single` | 单选 / 多选                |
| max_selections | 必要时 NOT NULL，默认 0                           | 多选时最多可选选项数；0 表示不限制       |
| start_at       | 可为空                                         | 开始时间（未到开始时间不生效/不展示）    |
| end_at         | 不为空                                         | 结束时间                   |
| created_by     | 可为空，外键 `users(id)`                          | 创建者（版主）                |
| created_at     | 不为空                                         |                        |

### 投票选项 (vote_options)

| 列名         | 约束                 | 描述/备注   |
|------------|--------------------|---------|
| id         | 主键                 | 随机 UUID |
| vote_id    | 不为空，外键 `votes(id)` | 所属投票    |
| content    | 不为空                | 选项内容    |
| sort_order | 不为空，默认 0           | 排序      |

### 投票记录 (vote_answers)

每个用户对一个投票最多投票一次；多选时一行一个被选选项。

| 列名         | 约束                        | 描述/备注   |
|------------|---------------------------|---------|
| id         | 主键                        | 随机 UUID |
| vote_id    | 不为空，外键 `votes(id)`        | 所属投票    |
| user_id    | 不为空，外键 `users(id)`        | 投票用户    |
| option_id  | 不为空，外键 `vote_options(id)` | 被选选项    |
| created_at | 不为空                       | 投票时间    |

> 唯一约束 `(vote_id, user_id, option_id)`；同一用户同一投票同一选项仅一条。

## Issue（议题）

类 GitHub Issues。公开 / 私有两种可见性。私有仅在**可见者**（创建者 + 有 `issue.private_read` 节点者）访问。无 assignee。

### 议题 (issues)

| 列名            | 约束                                            | 描述/备注        |
|---------------|-----------------------------------------------|--------------|
| id            | 主键                                            | 随机 UUID      |
| creator_id    | 不为空，外键 `users(id)`                            | 创建者          |
| title         | 不为空                                           | 标题           |
| body          | 可为空                                           | 正文（Markdown） |
| visibility    | NOT NULL，枚举 `public`/`private`，默认 `public`    | 可见性          |
| state         | NOT NULL，枚举 `open`/`closed`，默认 `open`         | 状态           |
| closed_reason | 可为空，枚举 `completed`/`duplicated`/`not_planned` | 关闭原因         |
| closed_by     | 可为空，外键 `users(id)`                            | 关闭者（版主）      |
| created_at    | 不为空                                           |              |
| updated_at    | 不为空                                           |              |
| closed_at     | 可为空                                           | 关闭时间         |

### 标签 (issue_labels)

标签由系统角色 `Admin` 创建 / 删除，`Helper`（协管）可分配给议题。

| 列名         | 约束                 | 描述/备注        |
|------------|--------------------|--------------|
| id         | 主键                 | 随机 UUID      |
| name       | NOT NULL，UNIQUE    | 标签名（如 `bug`） |
| color      | 可为空，默认 `#000000`   | 标签颜色（十六进制）   |
| created_by | 可为空，外键 `users(id)` | 创建者（管理）      |
| created_at | 不为空                |              |

### 议题-标签关联 (issue_label_records)

| 列名       | 约束                       | 描述/备注 |
|----------|--------------------------|-------|
| issue_id | 主键，外键 `issues(id)`       | 议题    |
| label_id | 主键，外键 `issue_labels(id)` | 标签    |

> `(issue_id, label_id)` 复合主键。

### 议题评论 (issue_comments)

| 列名         | 约束                  | 描述/备注   |
|------------|---------------------|---------|
| id         | 主键                  | 随机 UUID |
| issue_id   | 不为空，外键 `issues(id)` | 所属议题    |
| user_id    | 不为空，外键 `users(id)`  | 评论者     |
| content    | 不为空                 | 评论内容    |
| created_at | 不为空                 |         |

## 封禁 (bans)

简化设计：一个用户同一时刻至多一条记录。

| 列名           | 约束                             | 描述/备注                 |
|--------------|--------------------------------|-----------------------|
| id           | 主键                             | 随机 UUID               |
| user_id      | NOT NULL，UNIQUE，外键 `users(id)` | 被封禁的用户                |
| banned_until | 可为空                            | 解封时间（UTC），NULL 表示永久封禁 |
| reason       | 可为空                            | 封禁原因                  |
| operator_id  | 可为空，外键 `users(id)`             | 执行封禁的管理员              |
| created_at   | 不为空                            | 封禁时间                  |
| updated_at   | 不为空                            | 更新时间                  |

> 封禁命中判定：存在该用户记录且 `banned_until IS NULL OR banned_until > now()`。解封即删除记录。

## Yggdrasil 资源表

> 注：Yggdrasil 令牌（`accessToken`）**不存储于数据库**，由缓存实现（见 `yggdrasil/index.md`）。以下仅持久化角色、启动器会话与材质。

### 角色 (yggdrasil_profiles)

UUID 与名称全局唯一，名称可变。

| 列名         | 约束                      | 描述/备注                 |
|------------|-------------------------|-----------------------|
| id         | 主键                      | 角色 UUID（无符号）          |
| user_id    | NOT NULL，外键 `users(id)` | 所属账号                  |
| name       | NOT NULL，UNIQUE         | 角色名称                  |
| model      | NOT NULL，默认 `default`   | 材质模型 `default`/`slim` |
| created_at | 不为空                     |                       |
| updated_at | 不为空                     |                       |

### 启动器会话 (yggdrasil_auth_records)

启动器会话登录名（`email` 列）不可变更且全局唯一。每个启动器会话绑定且只能绑定一个角色（`profile_id`）。

| 列名           | 约束                                  | 描述/备注                              |
|--------------|-------------------------------------|------------------------------------|
| id           | 主键                                  | 启动器会话 ID（无符号 UUID）                 |
| user_id      | NOT NULL，外键 `users.id`              | 所属账号                               |
| email        | NOT NULL，UNIQUE                     | 启动器会话登录名（生成格式见 yggdrasil/index.md） |
| profile_id   | NOT NULL，外键 `yggdrasil_profiles.id` | 绑定的角色                              |
| raw_password | NOT NULL                            | 启动器会话认证凭据（随机无符号 UUID，长度可配置）        |
| created_at   | 不为空                                 |                                    |

### 角色材质 (yggdrasil_textures)

每个角色每类材质（SKIN/CAPE）至多一条。

| 列名           | 约束                                  | 描述/备注              |
|--------------|-------------------------------------|--------------------|
| id           | BIGINT 自增主键                         |                    |
| profile_id   | NOT NULL，外键 `yggdrasil_profiles.id` | 所属角色               |
| texture_type | NOT NULL，`skin`/`cape`              | 材质类型               |
| hash         | NOT NULL                            | 材质文件 hash（URL 文件名） |
| model        | 可为空                                 | 材质模型               |
| created_at   | 不为空                                 |                    |
| updated_at   | 不为空                                 |                    |
| UNIQUE       | `(profile_id, texture_type)`        | 每类至多一条             |

## 后台账户

后台采用独立鉴权，与主站账号体系**一对一**：一个系统角色为 `admin` 的用户，后台账户有且唯一。后台会话状态**不落库**（见 [缓存](cache.md#后台会话jwt)。

### 后台账户 (console_admins)

| 列名                 | 约束                                           | 描述/备注                                       |
|--------------------|----------------------------------------------|---------------------------------------------|
| id                 | 主键，外键 `users.id`，UNIQUE                      | 对应主站用户（系统角色为 admin）                      |
| role               | NOT NULL，枚举 `admin`/`super_admin`，默认 `admin` | 后台级别：普通管理 / 超级管理                         |
| password_hash      | 可为空                                          | 后台登录密码的 Bcrypt 哈希；为空表示处于「待设密」状态            |
| must_change_password | NOT NULL，默认 `true`                          | 待设密标记：首次登录须先经 `set-password` 设置强密码；被重置后置回 `true` |
| created_at         | 不为空                                          |                                             |
| updated_at         | 不为空                                          |                                             |

> - 用户被提升为 `admin`（后台「系统角色变更」为 Admin）时**不会自动生成密码**，而是创建处于**待设密**状态（`password_hash` 为空、`must_change_password = true`）的后台账户；首次进入后台须经 `POST /management/console/auth/set-password` 手动设置强密码（强度必须达标，且 Bcrypt 比对不能与主站密码一致）。SuperAdmin 可用 `reset-password` 将普通 Admin 重置回待设密状态。
> - 由后端终端 `/admin` 命令设置的为 `super_admin`，其后台密码由终端**随即生成**（见 [后端命令系统](backend.md)），SuperAdmin 的后台密码仅能由终端 `/admin` 控制。
> - 级别来源：提升为 `admin` 时默认 `admin`；`/admin` 命令设置为 `super_admin`。
> - 后台会话（JWT 的 `jti`）存于缓存，不在此表，见 [缓存](cache.md)。

## 站内通知与全站公告

### 站内通知 (notifications)

单用户通知。后台发送，用户读取并按需标记已读。

| 列名             | 约束                     | 描述/备注   |
|----------------|------------------------|---------|
| id             | 主键                     | 随机 UUID |
| target_user_id | NOT NULL，外键 `users.id` | 接收用户    |
| title          | 不为空                    | 标题      |
| content        | 不为空                    | 内容      |
| is_read        | NOT NULL，默认 `false`    | 是否已读    |
| sent_by        | 可为空，外键 `users.id`      | 发送后台操作者 |
| created_at     | 不为空                    |         |

### 全站公告 (announcements)

| 列名           | 约束                 | 描述/备注                |
|--------------|--------------------|----------------------|
| id           | 主键                 | 随机 UUID              |
| title        | 不为空                | 标题                   |
| content      | 不为空                | 内容                   |
| published    | NOT NULL，默认 `true` | 是否对外展示               |
| published_at | 可为空                | 发布时间                 |
| sent_by      | 可为空，外键 `users.id`  | 发送后台操作者（super_admin） |
| created_at   | 不为空                |                      |

## 审计日志

### 主站审计日志 (audit_logs)

记录主站所有需要权限节点的操作。

| 列名             | 约束                        | 描述/备注                                                                |
|----------------|---------------------------|----------------------------------------------------------------------|
| id             | 主键                        | 随机 UUID                                                              |
| operator_id    | 可为空，外键 `users.id`         | 操作者                                                                  |
| action         | 不为空                       | 取值为 `资源.动作`（如 `user.ban`、`user.role_change`、`identity.group_assign`） |
| target_user_id | 可为空，外键 `users.id`         | 操作目标用户                                                               |
| result         | 不为空，枚举 `success`/`denied` | 结果                                                                   |
| ip             | 可为空                       | 操作者 IP                                                               |
| payload        | 可为空，JSON                  | 动作相关参数                                                               |
| created_at     | 不为空                       | 日志产生时间                                                               |

### 后台审计日志 (console_audit_logs)

记录后台全部操作，额外记录浏览器信息。

| 列名         | 约束                         | 描述/备注                                                           |
|------------|----------------------------|-----------------------------------------------------------------|
| id         | 主键                         | 随机 UUID                                                         |
| admin_id   | 可为空，外键 `console_admins.id` | 后台操作者                                                           |
| action     | 不为空                        | 如 `console.reload`、`console.role_change`、`console.group_assign` |
| ip         | 可为空                        | 操作者 IP                                                          |
| user_agent | 可为空                        | 浏览器信息                                                           |
| payload    | 可为空，JSON                   | 动作相关参数                                                          |
| result     | 不为空，枚举 `success`/`denied`  | 结果                                                              |
| created_at | 不为空                        | 日志产生时间                                                          |

## 索引说明

- `users.username` 唯一索引；`users.email` 唯一索引（可空）。
- `oidc_records(user_id, provider_id)` 唯一索引；`oidc_records(issuer, subject)` **UNIQUE**。
- `identity_groups.name` 唯一索引。
- `user_groups(user_id, group_id)` 复合主键；`group_id` 索引。
- `prefix_presets.value` 唯一索引；`user_prefixes(user_id, prefix_id)` 复合主键，`prefix_id` 索引。
- `bans.user_id` UNIQUE。
- `yggdrasil_profiles.name` 唯一索引；`user_id` 索引。
- `yggdrasil_auth_records.email` 唯一索引；`user_id`、`profile_id` 索引。
- `yggdrasil_textures(profile_id, texture_type)` UNIQUE；`profile_id` 索引。
- `console_admins.user_id` UNIQUE。
- `vote_answers(vote_id, user_id, option_id)` 唯一索引；`vote_id`、`user_id` 索引。
- `issue_labels.name` UNIQUE。
- `issue_label_records(issue_id, label_id)` 复合主键；`label_id` 索引。
- `issue_comments(issue_id)` 索引。
- `notifications(target_user_id)`、`announcements(published_at)` 索引。
- `audit_logs(created_at)`、`console_audit_logs(created_at)` 索引。

> 外键 `ON DELETE CASCADE` 级联删除保证一致性；审计日志表不参与级联删除（保留历史）。