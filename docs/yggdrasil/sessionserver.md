# 端点：/yggdrasil/sessionserver

对应 [Yggdrasil-服务端技术规范](https://yushijinhun.github.io/authlib-injector/zh/Yggdrasil-%E6%9C%8D%E5%8A%A1%E7%AB%AF%E6%8A%80%E6%9C%AF%E8%A7%84%E8%8C%83.html)。

## 目录

- [会话部分](#会话部分)
  - [客户端进入服务器](#客户端进入服务器)
  - [服务端验证客户端](#服务端验证客户端)
- [角色部分](#角色部分)
  - [查询角色属性](#查询角色属性)

## 会话部分

1. 服务端向待进服的玩家发送 serverId，其可以被视为一个随机的字符串。
2. 客户端将该 serverId 连同自己的 accessToken 发送给 Yggdrasil 端，请求验证。
3. 服务端将 serverId 及玩家的名字发送给 Yggdrasil 端，查询玩家是否已完成验证。
4. 若玩家验证成功，则Yggdrasil端将玩家的 Profile返 回给服务端，其中包含材质信息和数字签名。
5. 服务端将该玩家的Profile分发给服内所有玩家。

该部分用于角色进入服务器时的验证。主要流程如下：

1. **Minecraft 服务端和 Minecraft 客户端** 共同生成一段字符串（`serverId`），其可以被认为是随机的
2. **Minecraft 客户端** 将 `serverId` 及令牌发送给 **Yggdrasil 服务端**（要求令牌有效）
3. **Minecraft 服务端** 请求 **Yggdrasil 服务端** 检查客户端会话的有效性，即客户端是否成功进行第 2 步

### 客户端进入服务器

`POST /yggdrasil/sessionserver/session/minecraft/join`

记录服务端发送给客户端的 `serverId`，以备服务端检查。

**请求**：
```json5
{
    "accessToken":"令牌的 accessToken",
    "selectedProfile":"该令牌绑定的角色的 UUID（无符号）",
    "serverId":"服务端发送给客户端的 serverId"
}
```

仅当 `accessToken` 有效，且 `selectedProfile` 与令牌所绑定的角色一致时，操作才成功。

服务端应记录以下信息：

* serverId
* accessToken
* 发送该请求的客户端 IP
* 实现时请注意：以上信息应记录在缓存中（可以是 Redis 数据库），且应该设置过期时间（如 30 秒）。 介于 `serverId` 的随机性，可以将其作为主键。

若操作成功，服务端应返回 HTTP 状态 `204 No Content`。

### 服务端验证客户端

`GET /yggdrasil/sessionserver/session/minecraft/hasJoined?username={username}&serverId={serverId}&ip={ip}`

检查客户端会话的有效性，即数据库中是否存在该 `serverId` 的记录，且信息正确。

**请求**：参数如下：

| 参数        | 值                                                                                                                                             |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| username  | 角色的名称                                                                                                                                         |
| serverId  | 服务端发送给客户端的 serverId                                                                                                                           |
| ip _（可选）_ | Minecraft 服务端获取到的客户端 IP，仅当 [`prevent-proxy-connections`](https://minecraft.gamepedia.com/Server.properties#prevent-proxy-connections) 选项开启时包含 |

`username` 需要与 `serverId` 所对应令牌所绑定的角色的名称相同。

响应格式：
```json5
{
	// ... 令牌所绑定角色的完整信息（包含角色属性及数字签名，格式见 yggdrasil/index.md#角色信息的序列化）
}
```

若操作失败，服务端应返回 HTTP 状态 `204 No Content`。


## 角色部分

该部分用于角色信息的查询。

### 查询角色属性

`GET /yggdrasil/sessionserver/session/minecraft/profile/{uuid}?unsigned={unsigned}`

查询指定角色的完整信息（包含角色属性）。

**请求**：参数如下：

请求参数：

| 参数              | 值                                             |
|-----------------|-----------------------------------------------|
| uuid            | 角色的 UUID（无符号）                                 |
| unsigned _（可选）_ | `true` 或 `false`。是否在响应中**不包含**数字签名，默认为 `true` |

响应格式：
```json5
{
	// ... 角色信息（包含角色属性。若 unsigned 为 false，还需要包含数字签名。格式见 yggdrasil/index.md#角色信息的序列化）
}
```

若角色不存在，服务端应返回 HTTP 状态 `204 No Content`。