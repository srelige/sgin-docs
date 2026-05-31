# 第 10 章：API Key 不需要 AuthRegistry

张三想接一个 API Key 认证。他问李四：“要不要做一个 AuthRegistry，像 DRF authentication classes 那样？”

李四说：“我们底层是 Gin，认证扩展就是 middleware。”

## 张三的低效想法

为了支持 API Key，再做一套注册表：

- 注册认证类
- 给 ViewSet 配认证类名称
- 再把认证类转成 handler 链

这会和 Gin middleware 重复。

## 李四的提醒

sgin 不做 AuthRegistry。

推荐方式：

```txt
API Key              Gin middleware
内部服务 token       Gin middleware
自定义 Bearer token  Gin middleware
签名认证             Gin middleware
租户认证             Gin middleware
```

内置 `Auth` 字段只表示使用 sgin 用户系统的 `JWTAuth()`。

## 如果还要接权限系统

自定义认证成功后，如果要复用 `LoadAccess()`、用户组、角色或动态路由权限，需要写入 sgin 能识别的 `user`。

最直接的方式是查到 `sgin.UserAccount` 后写入 Gin Context：

```txt
c.Set("user", user)
```

然后再串：

```txt
LoadAccess
RequireAnyGroup / RequireAnyRole / RequireRoutePermission
```

## 这一章记住什么

不要为 Gin 已经擅长的 middleware 组合再造一层认证注册表。sgin 保留 Gin 的扩展方式。
