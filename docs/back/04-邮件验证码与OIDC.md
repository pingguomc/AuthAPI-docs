# 后端开发指导（四）——邮件、验证码与 OIDC

> 语言无关，聚焦邮件验证码与 OIDC 集成的注意事项。

## 邮件与验证码

以下操作需要发送邮箱验证码：

- 邮箱注册；找回密码；修改密码；更换绑定邮箱（新邮箱收码）。

实现注意：

- **一次性**：使用后立即失效；**有时效**：过期即拒；**限频**：同一邮箱发送频繁则拒，防轰炸。
- 验证码及状态存缓存（键 `email:code:`，见文件三），TTL 与生成格式从业务配置读取。
- 校验时须确认**请求中的邮箱**与**发码时用的邮箱一致**，防止串号。
- 邮件通过 SMTP 发送；**未配置 SMTP 时进入开发模式**，把验证码打到日志（标记 `[DEV-MAIL]`）方便本地联调。生产必须配真实 SMTP。

## OIDC

采用 **Authorization Code Flow + PKCE(S256)**。支持两种入口：未登录时登录/注册、已登录时绑定 Provider。

- Provider 列表来自业务配置（`OIDC.toml`），数量不限，每项含标识、issuer、客户端 ID/密钥、JWKS、授权/Token 端点等。OIDC 有全局开关。

**发起授权/绑定时（暂存于会话缓存）**：

- 随机 `state`（防 CSRF）；随机 `code_verifier`（43~128 字符）；`code_challenge = base64url(sha256(code_verifier))`。
- 标记 `action`：`login` 或 `bind`。可选存 `redirect_uri` 作为回调后跳转目标。
- 回调时 Query 带 `code`、`state`、失败时带 `error`/`error_description`。

**回调验证步骤**：

1. 校验 `state` 匹配，否则拒绝。
2. 存在 `error` 参数 → 视为用户取消/失败，直接终止并跳转前端。
3. 用 `code` + `code_verifier` 调 `/token` 端点换令牌。
4. 验证 `id_token` 的 **签名、iss、aud、exp**。
5. 可选：调 `/userinfo` 补 claims。

**按 action 分流**：

- `login`：按 `(iss, sub)` 匹配用户——已存在直接登录；不存在自动注册 + 建绑定再登录。封禁用户不建会话。
- `bind`：校验当前已登录；该 Provider 已被其他用户绑定 → 拒绝；当前用户已绑 → 幂等成功不报错。

**与绑定约束联动**：OIDC 绑定与邮箱共享「至少一种登录方式」规则；OIDC 注册未设密码者至少留一个 OIDC 绑定（见文件二）。

> 前端交互（302 跳转等）不属于后端本文档范围，前端细节见端点文档。