# 错误码速查表（按 HTTP 状态码排序，error 字母序二级排序）

| HTTP状态码 |             error             | errorMessage（建议默认文案）              | 适用端点                                                                                                       |
|:-------:|:-----------------------------:|:----------------------------------|------------------------------------------------------------------------------------------------------------|
| **400** |   `CaptchaActionMismatch`  | 人机验证动作不匹配，请重新验证                   | 所有受人机验证保护的端点                                                                                               |
| **400** |        `CaptchaExpired`      | 人机验证已过期，请重新验证                     | 所有受人机验证保护的端点                                                                                               |
| **400** |        `CaptchaInvalid`      | 人机验证失败，请重试                        | 所有受人机验证保护的端点                                                                                               |
| **400** |        `CaptchaMissing`      | 该操作需要人机验证，请完成验证                   | 所有受人机验证保护的端点                                                                                               |
| **400** |      `EmailCodeExpired`       | 邮箱验证码已过期，请重新获取                    | `POST /user/change-password`、`PUT /user/email`                                                             |
| **400** |      `EmailMismatch`        | 邮箱与发验证码时不一致                       | `PUT /user/email`、`POST /email/code/set-email`                                            |
| **302** |       `IdTokenInvalid`        | 登录凭证校验失败，请重试                      | `GET /user/oidc/{providerId}/callback`                                                                     |
| **400** |     `InvalidDisplayName`      | 显示名称格式不正确                         | `POST /user/register`                                                                                      |
| **400** |        `InvalidEmail`         | 邮箱格式不正确                           | `POST /user/register`、`POST /email/code/*`                                                                 |
| **400** |     `InvalidOption`        | 选项不存在或不属于该投票                   | `POST /votes/{voteId}/answer` |
| **400** |     `InvalidPrefix`        | 前缀不在预设清单内                        | `PUT /management/users/{userId}/prefix` |
| **400** |      `InvalidEmailCode`       | 邮箱验证码错误                           | `POST /user/register`、`POST /user/change-password`、`PUT /user/email`                                       |
| **400** |       `InvalidRequest`        | 请求格式错误，请检查参数                      | 所有端点
| **400** |       `InvalidRole`        | 角色取值不合法                           | `POST /management/console/users/{userId}/roles` |
| **400** |     `ConfigParseError`     | 配置文件解析失败，本次改动未应用                 | `POST /management/console/reload` |
| **400** |       `VoteClosed`            | 投票已截止                             | `POST /votes/{voteId}/answer` |
| **400** |       `LastLoginMethod`       | 这是唯一的登录方式，不可解绑，请先绑定邮箱或其他 OIDC 提供商 | `DELETE /user/oidc/{providerId}`                                                                           |
| **302** |       `ProviderDenied`        | 用户取消了授权                           | `GET /user/oidc/{providerId}/callback`                                                                     |
| **400** |        `SamePassword`         | 新密码不能与旧密码相同                       | `POST /user/change-password`                                                                               |
| **302** |        `StateMismatch`        | 安全校验失败，请重试                        | `GET /user/oidc/{providerId}/callback`                                                                     |
| **302** |     `TokenExchangeFailed`     | 登录失败，请重试                          | `GET /user/oidc/{providerId}/callback`                                                                     |
| **400** |        `WeakPassword`         | 密码强度不足，需包含字母和数字且不少于6位             | `POST /user/register`、`POST /user/change-password`                                                         |
| **401** |     `InvalidCredentials`      | 邮箱或密码错误（或邮箱或验证码错误）                | `POST /user/login`                                                                                         |
| **401** |        `Unauthorized`         | 请先登录                              | 所有需要 Cookie 身份验证的端点                                                                                        |
| **403** |      `CaptchaRequired`       | 本次操作需要人机验证，请完成挑战                  | `onDemand` 模式下触发的端点                                                                                        |
| **403** |    `CaptchaScoreTooLow`      | 人机验证评分不足，请重试                      | 评分型 provider 的端点（预留）                                                                                       |
| **403** |          `Forbidden`          | 无权执行此操作                           | 所有需要特定角色的端点（含管理后台入口）                                                                                       |
| **403** |     `InvalidOldPassword`      | 旧密码错误                             | `POST /user/change-password`                                                                               |
| **403** |         `LoginLocked`         | 登录失败次数过多，请稍后再试                    | `POST /user/login`                                                                                         |
| **403** |        `OidcDisabled`         | OIDC 功能未启用                        | `GET /user/oidc/{providerId}/authorize`、`GET /user/oidc/{providerId}/bind`                                 |
| **403** |         `UserBanned`          | 该账号已被封禁                           | `POST /user/login`、`GET /user/oidc/{providerId}/callback`                                                  |
| **403** |       `ConsoleAdminDisabled`  | 该后台账户已被禁用                        | `POST /management/console/auth/login`                                                  |
| **404** |     `EmailNotRegistered`      | 该邮箱未注册                            | `POST /email/code/login`                                                                                   |
| **404** |      `ProviderNotFound`       | 不支持该 OIDC 提供商                     | `GET /user/oidc/{providerId}/*`                                                                            |
| **404** |      `ProviderNotBound`       | 未绑定该 OIDC 提供商                     | `DELETE /user/oidc/{providerId}`                                                                           |
| **404** |        `UserNotFound`         | 用户不存在                             | `GET /management/users/{userId}`、`POST /management/bans` |
| **404** |      `ProfileNotFound`        | 角色不存在                             | `GET/PATCH/DELETE /management/yggdrasil/profiles/{profileId}` |
| **404** |      `TextureNotFound`        | 材质不存在                             | `DELETE /management/yggdrasil/textures/{hash}` |
| **404** |        `GroupNotFound`        | 身份组不存在                           | `PATCH/DELETE /management/console/identity-groups/{groupId}`、`DELETE /management/console/users/{userId}/groups/{groupId}` |
| **404** |     `GroupNotAssigned`        | 该用户未分配此身份组                      | `DELETE /management/console/users/{userId}/groups/{groupId}` |
| **404** |  `NotificationNotFound`       | 通知不存在或不属于当前用户                 | `PUT /user/notifications/{id}/read` |
| **404** |       `VoteNotFound`          | 投票不存在                             | `GET/POST /votes/{voteId}`、`GET /votes/{voteId}/data` |
| **404** |      `IssueNotFound`          | 议题不存在或不可见                       | `GET/PATCH /issues/{issueId}`、`POST /issues/{issueId}/comments` |
| **404** |        `LabelNotFound`        | 标签不存在                             | `DELETE /management/console/labels/{labelId}` |
| **404** |      `PrefixNotFound`         | 前缀预设不存在                          | `DELETE /management/console/prefixes/{prefixId}` |
| **404** |         `BanNotFound`         | 未找到该用户的封禁记录                       | `GET/DELETE /management/bans/{userId}` |
| **409** |      `BanAlreadyExists`       | 该用户已有生效中的封禁记录                     | `POST /management/bans` |
| **409** |    `ProfileNameTaken`         | 角色名称已被占用                          | `PATCH /management/yggdrasil/profiles/{profileId}` |
| **409** |    `TextureInUse`             | 材质仍被角色引用，不可删除                    | `DELETE /management/yggdrasil/textures/{hash}` |
| **409** |    `GroupNameTaken`           | 身份组名称已被占用                         | `POST /management/console/identity-groups`、`PATCH /management/console/identity-groups/{groupId}` |
| **409** |    `GroupAlreadyAssigned`     | 该用户已分配此身份组                       | `POST /management/console/users/{userId}/groups` |
| **409** |     `LabelNameTaken`          | 标签名称已被占用                          | `POST /management/console/labels` |
| **409** |      `PrefixTaken`            | 前缀值已在预设中                          | `POST /management/console/prefixes` |
| **409** |   `EmailAlreadyRegistered`    | 该邮箱已被注册                           | `POST /user/register`、`POST /email/code/register`、`PUT /user/email`                                        |
| **409** |    `ProviderAlreadyBound`     | 该账号已绑定其他用户                        | `GET /user/oidc/{providerId}/callback`（bind 场景）                                                            |
| **429** |    `EmailCodeRateLimited`     | 验证码发送过于频繁，请稍后再试                   | `POST /email/code/*`                                                                                       |
| **429** |         `RateLimited`         | 操作过于频繁，请稍后再试                      | 所有端点（通用限流）                                                                                                 |
| **500** |        `InternalError`        | 服务器内部错误，请稍后重试                     | 所有端点                                                                                                       |
| **503** | `CaptchaServiceUnavailable` | 人机验证服务暂不可用，请稍后重试                  | 所有受人机验证保护的端点                                                                                               |

> 标注为 **`302`** 的 OIDC callback 错误并非 JSON 错误体，而是以 `302` 重定向跳转至前端回调页，并在 URL 查询参数中携带 `status=error`、`error`（错误码）与 `errorMessage`（URL 编码的人类可读描述）。详见 [GET /user/oidc/{providerId}/callback](EP-user.md#get-useroidcprovideridcallback)。