# 持久数据库表结构及模型参考

表名用小写+复数。时间一律采用 ISO 8601 UTC 字符串（`VARCHAR(32)`）存储，与 API 约定一致。

## 用户 (users)

| 列名            | 约束                                                   | 描述/备注     |
|---------------|------------------------------------------------------|-----------|
| id            | 主键                                                   | 随机的 UUID  |
| email         | 唯一，可为空                                               | 主站登录邮箱    |
| password_hash | 可为空                                                  | Bcrypt 哈希 |
| username      | 唯一，不为空，默认 `user_它的id`                                | 仅展示用      |
| display_name  |                                                      | 昵称        |
| role          | 枚举 `user`/`helper`/`moderator`/`admin`，不为空，默认 `user` | 系统角色      |
| last_login_at | 可为空                                                  | 最后登录时间    |
| created_at    | 不为空                                                  | 注册时间      |
| updated_at    | 不为空                                                  | 更新时间      |

> **说明**：用户是否被封禁、封禁到何时**不冗余存储**在 `users` 中（去掉了旧版的 `status` / `bannedUntil` 投影）。封禁状态一律由 `bans` 表即时计算得出，凡返回 `status` 的接口均给出**计算后的有效值**，前端无需自行比对时间。

## OIDC 记录 (oidc_records)

| 列名          | 约束                 |
|-------------|--------------------|
| id          | 自增 INT 主键          |
| user_id     | 不为空，外键 `users(id)` |
| provider_id | 不为空                |
| issuer      | 不为空                |
| subject     | 不为空                |
| username    | 可为空                |
| created_at  | 不为空                |
| updated_at  | 不为空                |

## 用户权限节点 (user_permissions)

系统角色（`users.role`）内建了默认权限节点集；本表记录**超出**内建集合的额外权限节点授予。（权限节点定义见配置文件，动态读取。）

| 列名              | 约束                 | 描述/备注                      |
|-----------------|--------------------|----------------------------|
| user_id         | 主键，外键 `users(id)`  | 被授予的用户                     |
| permission_node | 主键                 | 权限节点名，如 `management.users` |
| granted_by      | 可为空，外键 `users(id)` | 授予操作者（后台 SuperAdmin）       |
| granted_at      | 不为空                | 授予时间                       |

`(user_id, permission_node)` 唯一索引。

## 封禁 (bans)

简化设计：**一个用户同一时刻至多一条记录**。`users` 不再冗余 `status` 字段，封禁状态由本表即时计算。

| 列名           | 约束                             | 描述/备注                   |
|--------------|--------------------------------|-------------------------|
| id           | 主键                             | 随机 UUID                 |
| user_id      | NOT NULL，UNIQUE，外键 `users(id)` | 被封禁的用户                  |
| banned_until | 可为空                            | 解封时间（UTC），`NULL` 表示永久封禁 |
| reason       | 可为空                            | 封禁原因                    |
| operator_id  | 可为空，外键 `users(id)`             | 执行封禁的管理员                |
| created_at   | 不为空                            | 封禁时间                    |
| updated_at   | 不为空                            | 更新时间                    |

> 封禁命中判定：存在该用户的记录，且 `banned_until IS NULL OR banned_until > now()`。解封即删除记录。

## Yggdrasil 资源表

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

| 列名           | 约束                                   | 描述/备注                              |
|--------------|--------------------------------------|------------------------------------|
| id           | 主键                                   | 启动器会话 ID（无符号 UUID）                 |
| user_id      | NOT NULL，外键 `users(id)`              | 所属账号                               |
| email        | NOT NULL，UNIQUE                      | 启动器会话登录名（生成格式见 yggdrasil/index.md） |
| profile_id   | NOT NULL，外键 `yggdrasil_profiles(id)` | 绑定的角色                              |
| raw_password | NOT NULL                             | 启动器会话认证凭据（随机无符号 UUID，长度可配置）        |
| created_at   | 不为空                                  |                                    |

### 角色材质 (yggdrasil_textures)

每个角色每类材质（SKIN/CAPE）至多一条。

| 列名           | 约束                                   | 描述/备注              |
|--------------|--------------------------------------|--------------------|
| id           | BIGINT 自增主键                          |                    |
| profile_id   | NOT NULL，外键 `yggdrasil_profiles(id)` | 所属角色               |
| texture_type | NOT NULL，`skin`/`cape`               | 材质类型               |
| hash         | NOT NULL                             | 材质文件 hash（URL 文件名） |
| model        | 可为空                                  | 材质模型               |
| created_at   | 不为空                                  |                    |
| updated_at   | 不为空                                  |                    |
| UNIQUE       | `(profile_id, texture_type)`         | 每类至多一条             |

## 后台账户与会话

### 后台账户 (console_admins)

后台采用独立鉴权，与主站账号体系无关。密码由系统直接生成（强密码），不由管理员自行设置；最多两名 SuperAdmin 的约束由应用层保证。

| 列名            | 约束                                | 描述/备注      |
|---------------|-----------------------------------|------------|
| id            | 主键                                | 随机 UUID    |
| username      | NOT NULL，UNIQUE                   | 登录名        |
| password_hash | NOT NULL                          | 系统生成强密码的哈希 |
| role          | NOT NULL，枚举 `admin`/`super_admin` | 后台专属级别     |
| display_name  | 可为空                               | 展示名        |
| created_at    | 不为空                               |            |
| updated_at    | 不为空                               |            |

### 后台会话 (console_sessions)

| 列名            | 约束                               | 描述/备注           |
|---------------|----------------------------------|-----------------|
| id            | 主键                               | 随机 UUID         |
| admin_id      | NOT NULL，外键 `console_admins(id)` | 所属后台账户          |
| session_token | NOT NULL，UNIQUE                  | 会话凭据（哈希存储，不存明文） |
| expires_at    | 不为空                              | 会话过期时间          |
| created_at    | 不为空                              |                 |

## 审计日志

### 主站审计日志 (audit_logs)

记录主站所有需要权限节点操作的审计日志。

| 列名             | 约束                        | 描述/备注                                                  |
|----------------|---------------------------|--------------------------------------------------------|
| id             | 主键                        | 随机 UUID                                                |
| operator_id    | 可为空，外键 `users(id)`        | 操作者                                                    |
| action         | 不为空                       | 取值形如 `user.ban`、`user.role_change` 等（`资源.动作`，并对节点前缀匹配） |
| target_user_id | 可为空，外键 `users(id)`        | 操作目标用户                                                 |
| result         | 不为空，枚举 `success`/`denied` | 结果                                                     |
| ip             | 可为空                       | 操作者 IP                                                 |
| payload        | 可为空，JSON                  | 动作相关参数                                                 |
| created_at     | 不为空                       | 日志产生时间                                                 |

### 后台审计日志 (console_audit_logs)

记录后台全部操作，额外记录浏览器信息。

| 列名         | 约束                          | 描述/备注                                   |
|------------|-----------------------------|-----------------------------------------|
| id         | 主键                          | 随机 UUID                                 |
| admin_id   | 可为空，外键 `console_admins(id)` | 后台操作者                                   |
| action     | 不为空                         | 如 `console.reload`、`console.role_grant` |
| ip         | 可为空                         | 操作者 IP                                  |
| user_agent | 可为空                         | 浏览器信息                                   |
| payload    | 可为空，JSON                    | 动作相关参数                                  |
| result     | 不为空，枚举 `success`/`denied`   | 结果                                      |
| created_at | 不为空                         | 日志产生时间                                  |

## 索引说明

- `users(email)` 唯一索引（隐式）；`users.username` 唯一索引。
- `oidc_records(user_id, provider_id)` 唯一索引，防止同一用户绑定同一 provider 多次。
- `user_permissions(user_id, permission_node)` 主键（复合）。
- `bans.user_id` UNIQUE，自动建唯一索引。
- `yggdrasil_profiles(name)` 唯一索引；`user_id` 外键索引。
- `yggdrasil_auth_records(email)` 唯一索引；`user_id`、`profile_id` 外键索引。
- `yggdrasil_textures(profile_id, texture_type)` UNIQUE；`profile_id` 索引。
- `yggdrasil_tokens(auth_record_id)`、`profile_id` 索引；`expires_at` 索引（过期清理）。
- `console_sessions(session_token)` 唯一索引。
- `audit_logs(created_at)`、`console_audit_logs(created_at)` 索引，用于按时间范围筛选日志。

> 外键 `ON DELETE CASCADE` 级联删除保证一致性；审计日志表不参与级联删除（保留历史）。