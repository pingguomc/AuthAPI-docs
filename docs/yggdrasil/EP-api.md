# 材质和扩展 API端点

## 目录

- [端点：/yggdrasil/api](#端点yggdrasilapi)
  - [按名称批量查询角色](#按名称批量查询角色)
  - [材质上传](#材质上传)
- [端点：/yggdrasil](#端点yggdrasil)
  - [API 元数据获取](#api-元数据获取)
  - [端点：/yggdrasil/launcher-sessions](#端点yggdrasillauncher-sessions)
    - [创建启动器会话 (cookie身份验证)](#创建启动器会话-cookie身份验证)
    - [获取启动器会话列表 (cookie身份验证)](#获取启动器会话列表-cookie身份验证)
    - [获取启动器会话信息 (cookie身份验证)](#获取启动器会话信息-cookie身份验证)
    - [删除启动器会话 (cookie身份验证)](#删除启动器会话-cookie身份验证)
  - [端点：/yggdrasil/profiles](#端点yggdrasilprofiles)
    - [创建角色 (cookie身份验证)](#创建角色-cookie身份验证)
    - [获取角色列表 (cookie身份验证)](#获取角色列表-cookie身份验证)
    - [获取角色信息](#获取角色信息)
    - [删除角色 (cookie身份验证)](#删除角色-cookie身份验证)
    - [前端专用上传和删除材质 (cookie身份验证)](#前端专用上传和删除材质-cookie身份验证)

## 端点：/yggdrasil/api

### 按名称批量查询角色

`POST /yggdrasil/api/profiles/minecraft`

批量查询角色名称所对应的角色。

**请求**：
```json5
[
	"角色名称"
	// ,... 还可以有更多
]
```

服务端查询各个角色名称所对应的角色信息，并将其包含在响应中。不存在的角色不需要包含。响应中角色信息的先后次序无要求。

响应格式：
```json5
[
	{
		// 角色信息（注意：不包含角色属性。格式见 §角色信息的序列化）
	}
	// ,...（可以有更多）
]
```

**安全提示：** 为防止 CC 攻击，需要为单次查询的角色数目设置最大值，该值至少为 2。并对 IP 进行速率限制。

### 材质上传

```
PUT /yggdrasil/api/user/profile/{uuid}/{textureType}
DELETE /yggdrasil/api/user/profile/{uuid}/{textureType}
```

设置或清除指定角色的材质。

> 并非所有角色都可以上传皮肤和披风。要获取当前角色能够上传的材质类型，参见 [`uploadableTextures` 可上传的材质类型](./index.md#uploadabletextures-可上传的材质类型)。

**请求**：

| 参数          | 值                               |
|-------------|---------------------------------|
| uuid        | 角色的 UUID（无符号）                   |
| textureType | 材质类型，可以为 `skin`（皮肤）或 `cape`（披风） |

请求需要带上 HTTP 头部 `Authorization: Bearer {accessToken}` 进行认证。若未包含 Authorization 头或 accessToken 无效，则返回 `401 Unauthorized`。  
注意：被设置或清除的角色，必须是 accessToken 所绑定的角色，否则视为 accessToken 无效。

如果操作成功，则返回 `204 No Content`。

下面分别介绍 PUT 和 DELETE 这两个 HTTP 方法的用法：

#### PUT 上传材质

请求的 `Content-Type` 为 `multipart/form-data`，请求载荷由以下部分组成：

| 名称（name） | 内容                                                                                                               |
|----------|------------------------------------------------------------------------------------------------------------------|
| model    | **（仅用于皮肤）** 皮肤的材质模型，可以为 `slim`（细胳膊皮肤）或空字符串（普通皮肤）。                                                                |
| file     | 材质图像，`Content-Type` 须为 `image/png`。<br>建议客户端设置 `Content-Disposition` 中的 `filename` 参数为材质图像的文件名，这可以被验证服务器用作材质的备注。 |

如果操作成功，则返回 `204 No Content`。

#### DELETE 清除材质

清除材质后，该类型的材质将恢复为默认。

## 端点：/yggdrasil

### API 元数据获取

`GET /yggdrasil/`

响应格式：
```json5
{
	"meta":{
		// 服务端的元数据，内容任意
	},
	"skinDomains":[ // 材质域名白名单
		"域名匹配规则 1"
		// ,...
	],
	"signaturePublickey":"用于验证数字签名的公钥"
}
```

`signaturePublickey` 是 PEM 格式的公钥，用于验证角色属性的数字签名。其以 `-----BEGIN PUBLIC KEY-----` 开头，以 `-----END PUBLIC KEY-----` 结尾，中间允许出现换行符，但不允许出现其他空白字符（亦允许文末出现换行符）。

#### 材质域名白名单

Minecraft 仅会从白名单中的域名下载材质。如果材质 URL 的域名不在白名单中，则会出现 `Textures payload has been tampered with (non-whitelisted domain)` 错误。采用此机制的原因见 [MC-78491](https://bugs.mojang.com/browse/MC-78491)。

材质白名单默认包含 `.minecraft.net`、`.mojang.com` 两项规则，你可以设置 `skinDomains` 属性以添加额外的白名单规则。规则格式如下：

* 如果规则以 `.`（dot）开头，则匹配以这一规则结尾的域名。
    * 例如 `.example.com` 匹配 `a.example.com`、`b.a.example.com`，**不匹配** `example.com`。
* 如果规则**不以** `.`（dot）开头，则匹配的域名须与规则**完全相同**。
    * 例如 `example.com` 匹配 `example.com`，**不匹配** `a.example.com`、`eexample.com`。

#### `meta` 中的元数据

`meta` 中的内容没有强制要求，以下字段均为可选。

##### 服务端基本信息

| Key                   | Value    |
|-----------------------|----------|
| serverName            | 服务器名称    |
| implementationName    | 服务端实现的名称 |
| implementationVersion | 服务端实现的版本 |

##### 服务器网址

如果您需要在启动器中展示验证服务器首页地址、注册页面地址等信息，您可以在 `meta` 中添加一个 `links` 字段。

`links` 字段的类型是对象，其中可以包含：

| Key      | Value     |
|----------|-----------|
| homepage | 验证服务器首页地址 |
| register | 注册页面地址    |

##### 功能选项

> 以下带有 **_(advanced)_** 标注的字段为高级选项，通常情况下**不需要**设置。

| Key                                    | Value                                                                                                                                                                                                                                                                      |
|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| feature.non\_email\_login              | 布尔值，指示验证服务器是否支持使用邮箱之外的凭证登录（如角色名登录），默认为 false。<br>在本项目中此项实际上无效。                                                                                                                                                                                                             |
| feature.legacy\_skin\_api              | _(advanced)_ 布尔值，指示验证服务器是否支持旧式皮肤 API，即 `GET /skins/MinecraftSkins/{username}.png`。<br>当未指定或值为 false 时，authlib-injector 会使用内建的 HTTP 服务器在本地处理对该 API 的请求；若值为 true，请求将由验证服务器处理。<br>详情见 [README § 参数] 中的 `-Dauthlibinjector.legacySkinPolyfill` 选项。                             |
| feature.no\_mojang\_namespace          | _(advanced)_ 布尔值，是否禁用 authlib-injector 的 Mojang 命名空间（@mojang 后缀）功能，默认为 false。<br>详情见 [README § 参数] 中的 `-Dauthlibinjector.mojangNamespace` 选项。                                                                                                                              |
| feature.enable\_mojang\_anti\_features | _(advanced)_ 布尔值，是否开启 Minecraft 的 anti-features，默认为 false。<br>详情见 [README § 参数] 中的 `-Dauthlibinjector.mojangAntiFeatures` 选项。                                                                                                                                              |
| feature.enable\_profile\_key           | _(advanced)_ 布尔值，指示验证服务器是否支持 Minecraft 的消息签名密钥对功能（即多人游戏中聊天消息的数字签名），默认为 false。<br>开启后，Minecraft 将通过 `POST /minecraftservices/player/certificates` 这一 API 获取验证服务器颁发的密钥对。当未指定或为 false 时，Minecraft 将不会在聊天消息中包含数字签名。<br>详情见 [README § 参数] 中的 `-Dauthlibinjector.profileKey` 选项。 |
| feature.username_check                 | _(advanced)_ 布尔值，指示 authlib-injector 是否启用用户名验证功能，默认为 false。<br>开启后用户名中带有特殊字符的玩家将无法加入服务器，详情见 [README § 参数] 中的 `-Dauthlibinjector.usernameCheck` 选项。                                                                                                                         |

[README § 参数]: https://github.com/yushijinhun/authlib-injector#参数

#### 响应示例
```json5
{
    "meta": {
        "implementationName": "yggdrasil-mock-server",
        "implementationVersion": "0.0.1",
        "serverName": "yushijinhun's Example Authentication Server",
        "links": {
            "homepage": "https://skin.example.com/",
            "register": "https://skin.example.com/register"
        },
        "feature.non_email_login": true
    },
    "skinDomains": [
        "example.com",
        ".example.com"
    ],
    "signaturePublickey": "-----BEGIN PUBLIC KEY-----\nMIICIj...（省略）...EAAQ==\n-----END PUBLIC KEY-----\n"
}
```

### 端点：/yggdrasil/launcher-sessions

#### 创建启动器会话 (cookie身份验证)

`POST /yggdrasil/launcher-sessions`

**请求**：账号（启动器会话所有者）身份验证通过 Cookie 进行。

```json5
{
  "selectedProfileID": "角色 UUID（无符号）" // 必须是当前账号所拥有的角色，启动器会话生成就会绑定到这个角色。 
}
```

**响应**：
```json5
{
  "id":"ID（启动器会话ID）",
  "username":"即为启动器会话登录名",
  "password":"即为启动器会话认证凭据",
  "selectedProfile":{
    // ... 绑定的角色（格式见 §角色信息的序列化）。此项不得为空。
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
  "properties":[ // 属性（数组，每一元素为一个属性。可不包含任何属性，即为空数组。）
    { // 用户偏好语言（可选）
      "name":"preferredLanguage",
      "value":"zh_CN",
    }
  ]
}
```

**备注**：
* 每个账号能创建启动器会话的数量应该是有限并且可配置的，默认配置为 3。
* `启动器会话ID` 使用随机生成（Version 4），作为主键。
* `启动器会话登录名` 可以通过配置文件更改生成方式：`6位随机短无符号UUID@example.com`（可通过配置更改域名） 或 `"用户 displayName" + "__" + "角色名"`。eg: "pingguomc__pingguomc"，默认第一种。应该设置唯一约束。
* `即为启动器会话认证凭据` 为 `固定长度随机无符号UUID`，可以通过配置文件更改长度（6-12），默认 6。

#### 获取启动器会话列表 (cookie身份验证)

`GET /yggdrasil/launcher-sessions`

**请求**：无请求体，通过 cookie 确认账号身份。

**响应**：不分页
```json5
{
  "total": 1, // 总计数量
  "launcherSessions": [
    {
      "id":" ID（启动器会话ID）",
      "username":"即为启动器会话登录名"
    },
    {
      // 同上
    }
  ]
}
```

#### 获取启动器会话信息 (cookie身份验证)

`GET /yggdrasil/launcher-sessions/{launcherSessionID}`

**请求**：无请求体，此端点仅能查询本 cookie 账号拥有的启动器会话。试图查询其他人的则返回 `403`。

**响应**：与 [创建启动器会话 (cookie身份验证)](#创建启动器会话-cookie身份验证) 的响应完全一致。

#### 删除启动器会话 (cookie身份验证)

`DELETE /yggdrasil/launcher-sessions/{launcherSessionID}`

**请求**：无请求体，此端点仅能删除本 cookie 账号拥有的启动器会话。试图删除其他人的则返回 `403`。

**响应**：成功返回 `204`。后端应该删除与此启动器会话关联的全部 [令牌](index.md#令牌token)

### 端点：/yggdrasil/profiles

#### 创建角色 (cookie身份验证)

`POST /yggdrasil/profiles`

**请求**：账号（角色所有者）身份验证通过 Cookie 进行。
```json5
{
  "name":"角色名称"
}
```

**响应**：
```json5
{
  "id":"角色 UUID（无符号）",
  "name":"角色名称"
}
```

**备注**：UUID 和名称均为全局唯一。每个账号能创建角色的数量应该是有限并且可配置的，默认配置为 2。

#### 获取角色列表 (cookie身份验证)

`GET /yggdrasil/profiles`

**请求**：无请求体，通过 cookie 确认账号身份。

**响应**：不分页
```json5
{
  "total": 1, // 总计数量
  "profiles": [
    {
      "id":"角色 UUID（无符号）",
      "name":"角色名称",
    },
    {
      // 其他
    }
  ]
}
```

#### 获取角色信息

使用 [端点：/yggdrasil/sessionserver-角色部分-查询角色属性](EP-sessionserver.md#查询角色属性)。

#### 删除角色 (cookie身份验证)

`DELETE /yggdrasil/profiles/{id}`

**请求**：无请求体，此端点仅能删除本 cookie 账号拥有的角色。试图删除其他人的则返回 `403`。

**响应**：成功返回 `204`。

**备注**：删除之后，绑定在本角色的启动器会话和令牌均自动失效。


#### 前端专用上传和删除材质 (cookie身份验证)

```
PUT /yggdrasil/profiles/{uuid}/{textureType}
DELETE /yggdrasil/profiles/{uuid}/{textureType}
```

设置或清除指定角色的材质，专门给前端调用。

> 并非所有角色都可以上传皮肤和披风。要获取当前角色能够上传的材质类型，参见 [`uploadableTextures` 可上传的材质类型](./index.md#uploadabletextures-可上传的材质类型)。

**请求**：

| 参数          | 值                               |
|-------------|---------------------------------|
| uuid        | 角色的 UUID（无符号）                   |
| textureType | 材质类型，可以为 `skin`（皮肤）或 `cape`（披风） |

请求需要带上 Cookie 进行认证。若未包含，则返回 `401`，若试图操作不属于他的角色，返回 `403` 。

如果操作成功，则返回 `204 No Content`。

下面分别介绍 PUT 和 DELETE 这两个 HTTP 方法的用法：

#### PUT 上传材质

请求的 `Content-Type` 为 `multipart/form-data`，请求载荷由以下部分组成：

| 名称（name） | 内容                                                                                                            |
|----------|---------------------------------------------------------------------------------------------------------------|
| model    | **（仅用于皮肤）** 皮肤的材质模型，可以为 `slim`（细胳膊皮肤）或空字符串（普通皮肤）。                                                             |
| file     | 材质图像，`Content-Type` 须为 `image/png`。<br>建议设置 `Content-Disposition` 中的 `filename` 参数为材质图像的文件名，这可以被验证服务器用作材质的备注。 |

如果操作成功，则返回 `204 No Content`。

#### DELETE 清除材质

清除材质后，该类型的材质将恢复为默认。

