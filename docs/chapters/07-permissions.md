# 第 7 章：别到处写 if admin

张三写权限时，习惯在 handler 里判断：

```txt
如果用户不是 admin，就拒绝。
```

写到第五个接口，他已经复制了好几遍。

李四说：“权限是请求链上的事情，不要散落在业务代码里。”

## 张三的低效写法

在每个 handler 里判断用户组、角色和权限点：

- 重复
- 容易漏
- 错误响应不统一
- 后台配置权限也用不上

## 李四的提醒

sgin 的访问控制建议按顺序组合：

```txt
JWTAuth / Auth
LoadAccess
RequireAnyGroup / RequireAnyRole / RequireRoutePermission
handler
```

`LoadAccess` 会加载用户组、角色、权限点。动态路由权限按数据库里的 method/path 查权限点。

## 什么时候用哪种权限

```txt
用户是否登录                  Auth 或 JWTAuth
是否属于某个团队              RequireAnyGroup
是否拥有某个角色              RequireAnyRole
后台可配置接口权限            RequireRoutePermission
对象归属判断                  ObjectPermissions
列表数据范围收窄              QueryPermissions
代码内固定动作规则            ActionPermissions
```

## 这一章记住什么

认证和组织级权限优先用 middleware。对象级和查询范围再用 ViewSet 权限接口补充。
