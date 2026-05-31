# 第 7 章：别到处写 if admin

周四下午，张三开始给接口加权限。

他的第一反应很直接：在 handler 里取当前用户，判断是不是 admin。如果不是，就返回错误。

第一个接口这样写还行。第二个接口也还行。写到第五个接口时，张三发现自己复制了很多判断。更麻烦的是，有的接口要 admin，有的接口要 ops，有的接口要研发组，有的接口要动态权限点。每个 handler 的错误响应还不完全一致。

李四 review 时说：“你正在把权限写成业务代码的噪音。”

## 权限为什么不能散落

权限判断如果散落在 handler 里，会带来几个问题：

- 重复代码多。
- 很容易漏掉某个接口。
- 错误响应不统一。
- 后台动态权限配置无法复用。
- 测试需要逐个 handler 猜权限逻辑。
- 业务逻辑和访问控制混在一起。

李四让张三记住：认证和大部分组织级权限，应该发生在请求链上，而不是业务函数深处。

在 Gin 和 sgin 里，这条请求链就是 middleware。

## sgin 推荐的权限链

一个常见受保护接口，会经过这样的顺序：

```txt
JWTAuth 或 ViewSet Auth
LoadAccess
RequireAnyGroup / RequireAnyRole / RequireRoutePermission
handler 或默认 ViewSet 动作
```

每一步职责不同。

认证负责确认“你是谁”。

`LoadAccess` 负责把这个用户的用户组、角色、权限点加载出来。

`RequireAnyGroup` 判断是否属于某些用户组。

`RequireAnyRole` 判断是否拥有某些角色。

`RequireRoutePermission` 根据 method/path 查动态路由权限。

handler 最好只处理业务，不要重新写一遍这些横切逻辑。

## 用户组、角色、权限点的层次

张三一开始把 group、role、permission 混在一起。

李四用内部平台举例：

用户组更像组织归属，比如：

```txt
admin
ops
dev
finance
```

角色更像一组职责，比如：

```txt
asset_manager
user_auditor
report_reader
```

权限点更细，通常对应某个动作或能力，比如：

```txt
book.read
book.write
asset.sync
user.reset_password
```

动态路由权限把 HTTP method/path 映射到权限点。用户可以通过直接角色或用户组继承角色获得权限点。

admin 组是特殊组，通常直接放行。

## ViewSet 的 Auth 字段

张三问：“如果我在 ViewSet 上写 `Auth: []string{"all"}`，它到底做什么？”

李四说，它表示这个 ViewSet 的路由需要使用 sgin 用户系统的 JWT 认证。

常见配置有两类：

- `[]string{"all"}`：保护全部方法。
- `[]string{"get", "delete"}`：只保护指定 HTTP 方法。

不配置时，默认公开。

这只是认证，不等于授权。认证确认身份，授权判断这个身份能不能访问。

## LoadAccess 必须放在授权判断之前

张三有一次直接写了 `RequireAnyGroup("ops")`，却忘了 `LoadAccess()`。结果权限判断拿不到用户组。

李四让他记住顺序：

```txt
先认证
再 LoadAccess
再 RequireAnyGroup / RequireAnyRole / RequireRoutePermission
```

没有身份，加载不了访问控制。

没有加载访问控制，用户组和角色判断就没有数据。

这类顺序问题应该在路由注册处一眼能看出来，而不是藏在 handler 内部。

## 动态路由权限适合什么

动态路由权限适合后台可配置的接口权限。

比如系统管理员在 Admin UI 里配置：

```txt
GET  /books       -> book.read
POST /books       -> book.write
GET  /books/export -> book.export
POST /users/:id/reset-password -> user.reset_password
```

然后给角色绑定权限点，再把角色分配给用户或用户组。

这样业务代码不用为每个接口写固定判断。权限配置可以由后台数据驱动。

## ExtraActions 的权限路径

张三担心 dispatcher 会影响权限路径。

比如 `GET /books/export` 可能内部经过了详情路由入口。如果动态权限看到的是 `/books/:id`，那后台配置就会错。

sgin 已经处理这个问题。ExtraActions 的 dispatcher 会把业务 path 写进上下文，`RequireRoutePermission` 优先使用业务 path。

所以后台仍然按直观路径配置：

```txt
GET /books/export
```

而不是理解内部路由实现。

## ActionPermissions、ObjectPermissions、QueryPermissions

李四继续告诉张三：middleware 解决不了所有权限。

有些权限和 ViewSet 动作有关，比如只允许某些人执行 `destroy`。这可以用 ActionPermissions。

有些权限和具体对象有关，比如只能查看自己创建的记录。这是 ObjectPermissions。

有些权限和列表范围有关，比如运维只能看到自己负责机房的资产。这是 QueryPermissions。

它们的层次不同：

```txt
middleware          先判断能不能进入这个接口
ActionPermissions   判断能不能执行这个动作
ObjectPermissions   判断能不能访问这个对象
QueryPermissions    收窄列表查询范围
```

不要把所有规则都硬写成用户组判断。

## 错误语义也属于权限设计

张三以前权限失败随手返回一个错误。李四提醒他，权限错误要稳定。

认证失败应该是 401。比如没 token、token 无效、token 过期、账号禁用。

权限失败应该是 403。比如身份已经成立，但不在用户组、不具备角色、缺少动态权限点。

这部分第 8 章会详细讲。这里先记住：权限链不仅要挡住请求，也要给客户端稳定语义。

## 这一章的判断题

李四让张三写接口前先问：

1. 这个接口是否需要登录？
2. 是否需要加载用户组、角色、权限点？
3. 权限是固定用户组，还是固定角色？
4. 是否应该由后台动态配置 method/path 权限？
5. 是否还有对象归属或查询范围限制？
6. 这些判断是否应该在 handler 之前完成？

如果答案清楚，权限结构就不会散。

## 张三的笔记

张三写下：

不要到处写 `if admin`。认证、用户组、角色和动态路由权限优先放在 middleware 链上。对象级和查询范围权限再用 ViewSet 权限接口补充。业务 handler 只处理业务。
