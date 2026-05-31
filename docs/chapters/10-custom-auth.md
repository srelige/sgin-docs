# 第 10 章：API Key 不需要 AuthRegistry

第一周快结束时，张三接到一个系统对接需求。

另一个内部服务要调用 sgin 项目里的接口，但它不是人类用户，不走用户名密码登录。对方希望带一个 API Key。

张三想到 DRF 里的 authentication classes，问李四：“我们要不要做一个 AuthRegistry？注册 APIKeyAuth、BearerAuth、InternalTokenAuth，然后每个 ViewSet 配名字。”

李四回答得很快：“不用。我们底层是 Gin。”

## 不要重复 Gin 已经擅长的事

Gin 的 middleware 天然适合做认证扩展。

API Key 是 middleware。

内部服务 token 是 middleware。

自定义 Bearer token 是 middleware。

签名认证是 middleware。

租户识别也是 middleware。

如果 sgin 再做一套 AuthRegistry，就会出现重复抽象：先注册认证类，再把认证类翻译成 Gin handler 链，最后还是回到 middleware。

李四说：“框架应该补 Gin 没帮你收拢的部分，不应该盖住 Gin 已经很自然的部分。”

## sgin 内置 Auth 的含义

张三问：“那 ViewSet 里的 `Auth` 字段是什么？”

李四说，它表示使用 sgin 用户系统配套的 JWT 认证。

也就是说，`Auth` 是内置用户系统的便捷开关，不是通用认证注册表。

如果你使用 sgin 的用户登录、access token、refresh token，就可以通过 `Auth` 或 `JWTAuth()` 保护接口。

如果你要 API Key，就直接写 Gin middleware。

## API Key middleware 的形态

一个 API Key middleware 通常做这些事：

- 从 header 读取 key。
- 判断 key 是否存在。
- 校验 key 是否有效。
- 根据 key 找到调用方身份。
- 必要时写入 Gin Context。
- 成功后 `c.Next()`。
- 失败时返回 401。

它不需要 sgin 核心理解所有认证类型。

张三可以在路由上组合：

```txt
ApiKeyAuth
handler
```

或者在 ViewSet 上组合：

```txt
ApiKeyAuth
LoadAccess
RequireRoutePermission
默认动作
```

前提是这个 API Key 最终能映射到 sgin 能识别的用户或访问主体。

## 如果要复用用户组和角色

张三问：“API Key 校验通过后，能不能继续用 `LoadAccess()`、`RequireAnyGroup()` 这些？”

李四说，可以，但你要把认证结果接到 sgin 的上下文约定上。

如果 API Key 对应的是某个 `UserAccount`，middleware 校验成功后可以把用户写入 Gin Context：

```txt
c.Set("user", user)
```

后面的 `LoadAccess()` 就能加载这个用户的用户组、角色和权限点。

然后你可以继续使用：

```txt
RequireAnyGroup
RequireAnyRole
RequireRoutePermission
```

这条链路说明，自定义认证不需要 AuthRegistry，也能接入 sgin 权限系统。

## 如果不需要权限系统

有些内部接口只需要验证一个共享 token，不需要映射到具体用户。

这种情况下更简单：middleware 校验通过后直接放行，失败返回 401。后面不串 `LoadAccess()` 就行。

不要为了复用权限系统，强行给每个机器调用方创建用户。如果业务只需要服务级认证，就保持简单。

## 自定义 Bearer token

张三又问：“如果不是 API Key，而是另一个系统签发的 Bearer token 呢？”

李四说，仍然是 middleware。

middleware 解析 Authorization 头，校验 token 签名或 introspection，拿到调用方身份。之后是否写入 `user`，取决于是否要复用 sgin 用户权限。

这比 AuthRegistry 更直接，也更符合 Gin 项目常规写法。

## 多种认证如何组合

有些接口可能既允许用户 JWT，也允许内部 API Key。

李四建议不要把这种复杂性塞进框架配置，而是在业务 middleware 中明确表达策略：

- 先尝试用户 JWT。
- 再尝试 API Key。
- 任一成功则写入合适上下文。
- 全部失败返回 401。

这种策略往往和项目安全规则相关，不适合做成 sgin 核心默认行为。

## 认证和权限仍然要区分

自定义认证成功，只代表“身份成立”。

它不自动代表“有权限访问”。

如果这个身份还要受用户组、角色或动态路由权限约束，就继续串访问控制 middleware。否则就只是一个认证过的调用方。

这和第 8 章的 401/403 边界一致：

- API Key 缺失或无效，401。
- API Key 有效但缺少权限，403。

不要因为认证方式自定义，就破坏错误语义。

## 为什么不做 AuthRegistry 是有意选择

张三最后明白了：不做 AuthRegistry 不是能力缺失，而是设计取舍。

sgin 的目标不是把 Gin 包成另一种完全不同的框架。它保留 Gin middleware 的组合方式，让用户用最自然的 Go 方式扩展认证。

DRF 的 authentication classes 在 Python 和 DRF 的生态里很自然。sgin 站在 Gin 上，middleware 才是自然接口。

为了“看起来像 DRF”去复制 AuthRegistry，反而会让框架更重。

## 这一章的判断题

李四让张三以后遇到认证扩展先问：

1. 这是不是一个 Gin middleware 就能表达的认证？
2. 认证成功后是否需要映射到 sgin 用户？
3. 是否需要继续使用 `LoadAccess()` 和权限 middleware？
4. 失败时是否返回稳定的 401 错误？
5. 这个策略是否属于具体项目，而不是 sgin 核心？

多数情况下，答案会指向 middleware。

## 张三的笔记

张三写下：

API Key、自定义 Bearer、内部服务 token 都走 Gin middleware。sgin 内置 `Auth` 只服务自己的 JWT 用户系统。自定义认证如果要接权限链，就写入 `user`；不需要权限链，就保持 middleware 简洁。
