# 端点：/management/console

本端点全部需要身份验证,采用 [Cookie HttpOnly 会话](../../index.md#cookie-格式)。

访问本端点下的任意接口要求会话用户拥有每个节点指定的权限，否则返回 `403`。


