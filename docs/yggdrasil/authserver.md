# 端点：/yggdrasil/authserver

对应 [Yggdrasil-服务端技术规范 - 用户部分](https://yushijinhun.github.io/authlib-injector/zh/Yggdrasil-%E6%9C%8D%E5%8A%A1%E7%AB%AF%E6%8A%80%E6%9C%AF%E8%A7%84%E8%8C%83.html#%E7%94%A8%E6%88%B7%E9%83%A8%E5%88%86)。

## 登录 

`POST /authserver/authenticate`

使用密码进行身份验证，并分配一个新的令牌。

**请求**：

```json5
{
    "username":"即为启动器会话登录名",
    "password":"即为启动器会话认证凭据",
    "clientToken":"由客户端指定的令牌的 clientToken（可选）",
    "requestUser":false, // 是否在响应中包含用户信息，默认 false
    "agent":{
        "name":"Minecraft",
        "version":1
    }
}
```
若请求中未包含 `clientToken`，服务端应该随机生成一个无符号 UUID 作为 `clientToken`。但需要注意 `clientToken` 可以为任何字符串，即请求中提供任何 `clientToken` 都是可以接受的，不一定要为无符号 UUID。

对于令牌要绑定的角色：若用户没有任何角色，则为空；若用户仅有一个角色，那么通常绑定到该角色；若用户有多个角色，通常为空，以便客户端进行选择。也就是说如果绑定的角色为空，则需要客户端进行角色选择。

**响应**：
```json5
{
    "accessToken":"令牌的 accessToken",
    "clientToken":"令牌的 clientToken",
    "availableProfiles":[ // 启动器会话信息可用角色列表
        // ,... 每一项为一个角色（格式见 §角色信息的序列化）
    ],
    "selectedProfile":{
        // ... 绑定的角色，若为空，则不需要包含（格式见 §角色信息的序列化）
    },
    "user":{
        // ... 用户信息->启动器会话信息（仅当请求中 requestUser 为 true 时包含，格式见 用户信息的序列化 ——> 启动器会话信息的序列化）
    }
}
```

安全提示： 该 API 可以被用于密码暴力破解，应受到速率限制。限制应针对用户，而不是客户端 IP。

**这部分内容没有计划好。待计划。**
```markdown
使用角色名称登录
除使用邮箱登录外，验证服务器还可以允许用户使用角色名称登录。要实现这一点，验证服务器需要进行以下工作：

将 API 元数据中的 feature.non_email_login 字段设置为 true。（见 API 元数据获取§功能选项）
接受在登录接口中使用角色名称作为 username 参数。
当用户使用角色名称登录时，验证服务器应自动将令牌绑定到相应角色，即上文响应中的 selectedProfile 应为用户登录时所用的角色。

这种情况下，如果用户拥有多个角色，那么他可以省去选择角色的操作。考虑到某些程序不支持多角色（例如 Geyser），还可以通过上述方法绕过角色选择。
```

## 刷新

`POST /authserver/refresh`

吊销原令牌，并颁发一个新的令牌。

**请求**：

```json5
{
    "accessToken":"令牌的 accessToken",
    "clientToken":"令牌的 clientToken（可选）",
    "requestUser":false, // 是否在响应中包含用户信息->启动器会话信息，默认 false
    "selectedProfile":{
        // ... 要选择的角色（可选，格式见 §角色信息的序列化）
    }
}
```

当指定 `clientToken` 时，服务端应检查 `accessToken` 和 `clientToken` 是否有效，否则只需要检查 `accessToken`。

颁发的新令牌的 `clientToken` 应与原令牌的相同。

如果请求中包含 `selectedProfile`，那么这就是一个选择角色的操作。此操作要求原令牌所绑定的角色为空，而新令牌则将绑定到 `selectedProfile` 所指定的角色上。如果不包含 `selectedProfile`，那么新令牌所绑定的角色和原令牌相同。

刷新操作在令牌暂时失效时依然可以执行。若请求失败，原令牌依然有效。

**响应**：
```json5
{
    "accessToken":"新令牌的 accessToken",
    "clientToken":"新令牌的 clientToken",
    "selectedProfile":{
        // ... 新令牌绑定的角色，若为空，则不需要包含（格式见 §角色信息的序列化）
    },
    "user":{
        // ... 用户信息->启动器会话信息（仅当请求中 requestUser 为 true 时包含，格式见 §用户信息的序列化 ——> 启动器会话信息的序列化）
    }
}
```

## 验证令牌

`POST /authserver/validate`

检验令牌是否有效。

**请求**：
```json5
{
    "accessToken":"令牌的 accessToken",
    "clientToken":"令牌的 clientToken（可选）"
}
```

当指定 `clientToken` 时，服务端应检查 `accessToken` 和 `clientToken` 是否有效，否则只需要检查 `accessToken` 。

若令牌有效，服务端应返回 HTTP 状态 `204 No Content`，否则作为令牌无效的异常情况处理。

## 吊销令牌

吊销给定令牌。

**请求**：
```json5
{
    "accessToken":"令牌的 accessToken",
    "clientToken":"令牌的 clientToken（可选）"
}
```

服务端只需要检查 `accessToken`，即无论 `clientToken` 为何值都不会造成影响。

无论操作是否成功，服务端应返回 HTTP 状态 `204 No Content`。

## 登出

`POST /authserver/signout`

吊销用户的所有令牌。

**请求**：
```json5
{
    "username":"启动器会话登录名",
    "password":"启动器会话认证凭据"
}
```

若操作成功，服务端应返回 HTTP 状态 `204 No Content`。

安全提示： 该 API 也可用于判断密码的正确性，因此应受到和登录 API 一样的速率限制。

