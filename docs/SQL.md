# 持久数据库表结构及模型参考

表名用小写+复数。

## User (users) [用户 / 账号]

| 列名            | 约束                                                       | 描述/备注   |
|---------------|----------------------------------------------------------|---------|
| id            | 主键                                                       | 随机的UUID |
| email         | 唯一                                                       | 可以为空    |
| password_hash |                                                          | 可以为空    |
| username      | 唯一，不为空，默认"user_它的id"                                     | 只是用户名而已 |
| display_name  |                                                          | 仅展示使用   |
| role          | 枚举("user","helper","moderator","admin")，<br>不为空，默认"user" | 系统角色    |
| perfix        |                                                          | 预留字段，无用 |
| last_login_at |                                                          | 最后登录时间  |
| created_at    | 不为空                                                      | 注册时间    |
| updated_at    | 不为空                                                      | 更新时间    |

## OIDC Record (oidc_records) [第三方认证记录]

| 列名          | 约束                   | 描述/备注 |
|-------------|----------------------|-------|
| id          | 自增INT主键              |       |
| user_id     | 不为空，外键约束，`users(id)` |       |
| provider_id | 不为空                  |       |
| issuer      | 不为空                  |       |
| subject     | 不为空                  |       |
| username    |                      |       |
| created_at  | 不为空                  |       |
| updated_at  | 不为空                  |       |

## Ban (bans) [封禁记录-已废弃]

| 列名           | 约束                        | 描述/备注                   |
|--------------|---------------------------|-------------------------|
| id           | 自增INT主键                   |                         |
| user_id      | 不为空，外键约束 `users(id)`，唯一约束 | 被封禁的用户                  |
| banned_until | 可为空                       | 封禁到期时间（UTC），NULL 表示永久封禁 |
| reason       | 可为空                       | 封禁原因文字描述                |
| operator_id  | 可为空，外键约束 `users(id)`      | 执行封禁操作的管理员用户ID          |
| created_at   | 不为空                       | 封禁创建时间（UTC）             |

## Audit Log (audit_logs) [审计日志-已废弃]

| 列名             | 约束                   | 描述/备注                                             |
|----------------|----------------------|---------------------------------------------------|
| id             | 主键                   | 随机的UUID                                           |
| operator_id    | 可为空，外键约束 `users(id)` | 操作人用户ID                                           |
| action         | 不为空                  | 操作类型，例如 `USER_LOGIN`, `ROLE_GRANT`, `POST_DELETE` |
| target_user_id | 可为空，外键约束 `users(id)` | 操作目标用户ID（若有）                                      |
| result         | 不为空                  | 操作结果，取值 `SUCCESS`, `FAIL`, `DENIED`               |
| ip             | 可为空                  | 发起请求的客户端IP                                        |
| payload        | 可为空                  | 额外信息，JSON 格式存储                                    |
| created_at     | 不为空                  | 日志产生时间（UTC）                                       |

## 索引说明

为了提高查询效率，已创建以下索引：

- `idx_oidc_records_user_id`：`oidc_records(user_id)`
- `idx_oidc_records_user_provider`：`oidc_records(user_id, provider_id)` 唯一索引，防止同一用户绑定同一 provider 多次
- `idx_audit_logs_created_at`：`audit_logs(created_at)`，用于按时间范围快速筛选日志
- 注意：`bans` 表的 `user_id` 已有唯一约束，自动创建唯一索引；主键默认自带索引。

> 外键使用 `ON DELETE CASCADE`（级联删除）保证数据一致性，但审计日志不删除。
