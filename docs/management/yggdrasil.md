# 端点：/management/yggdrasil

Yggdrasil 管理端点。从 `/management` 索引拆出，见 [management 索引](index.md)。

管理 Yggdrasil 角色与材质资源，数据模型见 [SQL.md](../SQL.md) 的 `yggdrasil_*` 表。

## 权限

本文件涉及的节点及默认最低系统角色：

| 权限节点 | 说明 | 默认最低系统角色 |
|-----------|------|-----------|
| `management.yggdrasil.profiles` | 角色管理 | `Moderator` |
| `management.yggdrasil.textures` | 材质管理 | `Moderator` |

> 去除了旧的“启动器会话管理”与“令牌管理”；令牌不落库（见 [缓存](../cache.md)），故不在此提供管理端点。

## 目录

- [角色](#角色)
  - [GET /management/yggdrasil/profiles](#get-managementyggdrasilprofiles)
  - [GET /management/yggdrasil/profiles/{profileId}](#get-managementyggdrasilprofilesprofileid)
  - [PATCH /management/yggdrasil/profiles/{profileId}](#patch-managementyggdrasilprofilesprofileid)
  - [DELETE /management/yggdrasil/profiles/{profileId}](#delete-managementyggdrasilprofilesprofileid)
- [材质](#材质)
  - [GET /management/yggdrasil/textures](#get-managementyggdrasiltextures)
  - [DELETE /management/yggdrasil/textures/{hash}](#delete-managementyggdrasiltextureshash)

---

## 角色

### GET /management/yggdrasil/profiles

角色列表。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**请求参数**：`userId`（可选）、`page` / `pageSize`。

**响应**：`200`：

```json5
{
  "total": 30,
  "page": 1,
  "pageSize": 20,
  "items": [
    {
      "id": "7b3f0c2f-...",
      "userId": "be081dbc-...",
      "name": "Steve",
      "model": "slim",
      "createdAt": "2026-08-08T10:30:00Z",
      "updatedAt": "2026-08-08T10:30:00Z"
    }
  ]
}
```

### GET /management/yggdrasil/profiles/{profileId}

角色详情。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**响应**：

```json5
{
  "id": "7b3f0c2f-...",
  "userId": "be081dbc-...",
  "name": "Steve",
  "model": "slim",
  "textures": [
    { "textureType": "skin", "hash": "e051c27e...", "model": "slim" }
  ],
  "createdAt": "2026-08-08T10:30:00Z",
  "updatedAt": "2026-08-08T10:30:00Z"
}
```

**备注**：不存在返回 `404`，`error` 为 `ProfileNotFound`。

### PATCH /management/yggdrasil/profiles/{profileId}

角色改名（名称全局唯一）。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**请求**：

```json5
{
  "name": "SteveNew"
}
```

**响应**：`200`，返回更新后的角色信息。

**后端处理**：改名后，绑定该角色的令牌（缓存）应标为**暂时失效**，令启动器刷新令牌以获取新名称。

**备注**：名称已存在返回 `409`，`error` 为 `ProfileNameTaken`。

### DELETE /management/yggdrasil/profiles/{profileId}

删除角色。删除后其材质一并清理。

**权限**：`management.yggdrasil.profiles`，默认最低系统角色 `Moderator`。

**请求**：无请求体。

**响应**：`204`。

**备注**：不存在返回 `404`，`error` 为 `ProfileNotFound`。

---

## 材质

### GET /management/yggdrasil/textures

材质列表（默认孤儿材质：引用的角色已删除或文件无记录）。

**权限**：`management.yggdrasil.textures`，默认最低系统角色 `Moderator`。

**请求参数**：`profileId`（可选）、`orphanOnly`（默认 `true`）、`page` / `pageSize`。

**响应**：`200`：

```json5
{
  "total": 8,
  "page": 1,
  "pageSize": 20,
  "items": [
    { "hash": "e051c27e...", "textureType": "skin", "profileId": null, "orphan": true }
  ]
}
```

### DELETE /management/yggdrasil/textures/{hash}

删除材质文件（按 hash）。仅允许删除孤儿材质。

**权限**：`management.yggdrasil.textures`，默认最低系统角色 `Moderator`。

**后端处理**：若材质仍被角色引用则拒绝。

**响应**：
- 成功：`204`。
- 材质被引用：`409`，`error` 为 `TextureInUse`。
- 不存在：`404`，`error` 为 `TextureNotFound`。