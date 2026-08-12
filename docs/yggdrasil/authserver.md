# 端点：/yggdrasil/authserver

对应 [Yggdrasil-服务端技术规范 - 用户部分](https://yushijinhun.github.io/authlib-injector/zh/Yggdrasil-%E6%9C%8D%E5%8A%A1%E7%AB%AF%E6%8A%80%E6%9C%AF%E8%A7%84%E8%8C%83.html#%E7%94%A8%E6%88%B7%E9%83%A8%E5%88%86)。

## 目录

- [登录](#登录)
- [刷新](#刷新)
- [验证令牌](#验证令牌)
- [吊销令牌](#吊销令牌)
- [登出](#登出)

## 登录

`POST /yggdrasil/authserver/authenticate`

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

对于每个启动器会话，都有且有唯一一个令牌要绑定的角色。这样直接绕过了角色选择，而不需要采取角色名登录的方式，避免了某些程序不支持多角色选择（例如 Geyser）。

**响应**：
```json5
{
    "accessToken":"令牌的 accessToken",
    "clientToken":"令牌的 clientToken",
    "availableProfiles":[ // 启动器会话信息可用角色列表
        // ,... 每一项为一个角色（格式见 §角色信息的序列化）。实际应当有且只有一个角色。
        // 下面为示例
      {
        "id":"角色 UUID（无符号）",
        "name":"角色名称",
        "properties":[ // 角色的属性（数组，每一元素为一个属性）（仅在特定情况下需要包含）
          { // 一项属性
            "name":"属性的名称",
            "value":"属性的值",
            "signature":"属性值的数字签名（仅在特定情况下需要包含）"
          }
          // ,...（可以有更多）
        ]
      }
    ],
    "selectedProfile":{
        // ... 绑定的角色（格式见 §角色信息的序列化）。即为上述角色，此项实际上不得为空。
        // 下面为示例
      "id":"角色 UUID（无符号）",
      "name":"角色名称",
      "properties":[ // 角色的属性（数组，每一元素为一个属性）（仅在特定情况下需要包含）
        { // 一项属性
          "name":"属性的名称",
          "value":"属性的值",
          "signature":"属性值的数字签名（仅在特定情况下需要包含）"
        }
        // ,...（可以有更多）
      ]
    },
    "user":{
        // ... 启动器会话信息（仅当请求中 requestUser 为 true 时包含，格式见 §启动器会话信息的序列化）
        // 下面是示例
      "id":" ID（启动器会话ID）",
      "properties":[ // 属性（数组，每一元素为一个属性。可不包含任何属性，即为空数组。）
        { // 用户偏好语言（可选）
          "name":"preferredLanguage",
          "value":"zh_CN",
        }
      ]
    }
}
```

安全提示： 该 API 可以被用于密码暴力破解，应受到速率限制。限制应针对用户，而不是客户端 IP。

### 使用角色名称登录

API 元数据中的 feature.non_email_login 字段实际上无效。（见 API 元数据获取§功能选项）

## 刷新

`POST /yggdrasil/authserver/refresh`

吊销原令牌，并颁发一个新的令牌。

**请求**：

```json5
{
    "accessToken":"令牌的 accessToken",
    "clientToken":"令牌的 clientToken（可选）",
    "requestUser":false // 是否在响应中包含用户信息（启动器会话信息），默认 false
}
```

当指定 `clientToken` 时，服务端应检查 `accessToken` 和 `clientToken` 是否有效，否则只需要检查 `accessToken`。

颁发的新令牌的 `clientToken` 应与原令牌的相同。

请求中不包含 `selectedProfile`。新令牌所绑定的角色与原令牌相同（即该启动器会话绑定的角色）。

刷新操作在令牌暂时失效时依然可以执行。若请求失败，原令牌依然有效。

**响应**：
```json5
{
    "accessToken":"新令牌的 accessToken",
    "clientToken":"新令牌的 clientToken",
    "selectedProfile":{
        // ... 新令牌绑定的角色（格式见 §角色信息的序列化）。此项不得为空。
    },
    "user":{
        // ... 启动器会话信息（仅当请求中 requestUser 为 true 时包含，格式见 §启动器会话信息的序列化）
    }
}
```

## 验证令牌

`POST /yggdrasil/authserver/validate`

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

`POST /yggdrasil/authserver/invalidate`

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

`POST /yggdrasil/authserver/signout`

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

