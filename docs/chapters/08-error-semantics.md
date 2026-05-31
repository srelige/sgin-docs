# 第 8 章：前端为什么分不清 401 和 403

前端同事问张三：“这个接口到底是要我重新登录，还是用户没权限？”

张三一看，所有错误都差不多。

李四说：“认证失败和权限失败必须稳定区分。”

## 张三的低效写法

不同地方随手返回错误：

- 没 token 返回 403
- 没权限也返回 401
- token 过期和权限不足用同一个 code

客户端就无法稳定处理。

## 李四的提醒

sgin 固定边界：

```txt
401：身份没有成立
403：身份成立，但权限不足
```

认证类错误包括：

- `authentication_required`
- `invalid_authorization`
- `invalid_token`
- `token_expired`
- `invalid_token_type`
- `account_disabled`

权限类错误包括：

- `permission_denied`
- `admin_required`
- `group_required`
- `role_required`
- `route_permission_required`
- `route_permission_not_configured`

## 这一章记住什么

HTTP status 给粗粒度语义，响应里的稳定 `error` code 给客户端分支和日志检索。
