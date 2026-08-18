# 请求速率限制

采用专门的速率配置文件（`rate-limit.toml`），用于填写每个速率限制的数。格式统一为 `{次数} / {秒数}`，读作 X 秒 X 次，例如 `5/10` 表示 10 秒内最多 5 次。  
{次数} 为可配置的值，文档填写的数字为默认值。

## 全局读取接口

全部无需身份认证的仅 GET 接口，`25/1`（每个 IP）。

## 全局需 Cookie 认证的端点

除邮件端点 (/email) 外，采取 `用户ID` 和 `IP地址` 限速策略，即：`25/1`（每个用户ID），且 `50/1`（每个 IP），二者必须同时满足。

## 登录（包含OIDC登录）、注册等无需 Cookie 认证的端点

采取 `IP地址` 限速策略，即：`5/10` 且 `60/3600`（每个 IP）。

## Yggdrasil 端点

`/yggdrasil/authserver/authenticate`、`/yggdrasil/authserver/signout` 则 `1/1`（每个启动器会话）。

`/yggdrasil/api/profiles/minecraft` 则 `5/1`（每个 IP）。

`PATCH /yggdrasil/profiles/{id}`（角色改名）则 `1/31536000`（每个 profileId，默认一年最多改一次）。

其他需要 `accessToken` 认证的，`10/1`（每个 accessToken）。

## 账号登录失败锁定（LoginLocked）

针对**单个账号**的登录失败计数（缓存键 `login_attempts:{userId}`，见 [缓存](cache.md)），独立于 IP 维度限流生效：

- 默认：一个账号连续失败 `5` 次 / `15` 分钟，则锁定该账号 `15` 分钟；锁定期间 `POST /user/login` 返回 `403 LoginLocked`。
- 阈值与锁定期均可通过 `rate-limit.toml` 配置；登录成功后或锁定结束后清零。
- 独立于 IP 维度限流与人机验证开关，任何人机验证关闭都不会放宽该锁定。

## 邮件端点

### 无 Cookie 认证

1. `1/10`（每个 IP）。
2. `1/60` 且 `10/3600`（每个邮箱）。
3. `1/60` （每个 `IP` + `邮箱` 组合）。（优先级最高）

### 有 Cookie 认证

若该邮箱是用户绑定的邮箱 (`/email/code/change-password`)，则 `1/10`（每个用户ID）。否则 `1/30`（每个用户ID），且 `1/30`（每个邮箱），二者必须同时满足。

## 其他

OIDC callback (`/user/oidc/*/callback`) 端点不使用上述限速规则，因为 state 为一次性使用。