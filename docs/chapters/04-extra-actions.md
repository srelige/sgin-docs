# 第 4 章：reset-password 到底放哪

周三下午，张三开始做用户模块。

基础用户 CRUD 已经用 `ModelViewSet` 注册好了。列表、详情、创建、编辑都很顺。接着产品补了一个需求：管理员可以重置某个用户的密码。

张三想了一个 URL：

```txt
POST /reset-user-password
```

他觉得这个路径很直白。handler 里接收用户 ID，然后重置密码。

李四看完后问：“这个动作属于谁？”

张三说：“属于用户。”

李四继续问：“那为什么不挂在 `/users` 下面？”

## 资源动作散落的问题

张三的第一版不是不能跑，但会让路由慢慢失去组织。

一开始只是：

```txt
POST /reset-user-password
```

后来又会出现：

```txt
POST /sync-asset
GET  /export-books
POST /close-ticket
POST /enable-account
POST /disable-account
```

这些动作看起来都是普通路由，彼此之间没有资源归属。新人看路由表时，很难判断它们应该和哪个 ViewSet 一起维护，也很难判断它们继承了哪些认证和权限规则。

李四说：“资源内动作应该跟资源聚合。否则 ViewSet 只管 CRUD，业务动作到处飘，最后还是散。”

## DRF 的影子和 sgin 的取舍

张三以前看过 DRF，知道 ViewSet 里有 `@action`。比如：

```txt
/users/:id/reset-password
/books/hot
```

sgin 没有 Python 装饰器，也不需要照搬 DRF 的函数名机制。但资源动作聚合这个想法是有价值的。

于是 sgin 提供轻量的 `ExtraActions`。

它不做复杂动作元编程，只让你在 ViewSet 上声明额外路由：这个动作是什么 method，path 是什么，是 detail 还是 collection，有没有额外 middleware，handler 是谁。

## Detail 到底是什么意思

张三第一次看到 `Detail` 时有点困惑。

李四解释：`Detail` 不是“返回详情”的意思，而是“这个动作是否绑定到某一个具体对象”。

如果动作需要一个资源 ID，它就是 detail action：

```txt
POST /users/:id/reset-password
POST /assets/:id/sync
POST /tickets/:id/close
```

如果动作面向整个集合，它就是 collection action：

```txt
GET  /books/export
GET  /books/hot
POST /assets/import
```

所以重置某个用户密码，应该是 `Detail: true`。

## 张三的 ExtraActions 版本

张三把散落路由收回到用户 ViewSet 里：

```go
app.Register(&sgin.ModelViewSet[User, uint]{
    BasePath: "/users",
    Serializer: sgin.ModelSerializer[User]{
        ReadFields:  []string{"id", "username", "email"},
        WriteFields: []string{"username", "email"},
    },
    ExtraActions: []sgin.ExtraAction{
        {
            Method: "post",
            Path:   "reset-password",
            Detail: true,
            Middlewares: []gin.HandlerFunc{
                app.RequireAnyGroup("admin", "ops"),
            },
            Handler: handlers.ResetPassword,
        },
    },
})
```

最终路由是：

```txt
POST /users/:id/reset-password
```

它继承用户 ViewSet 的整体配置，又能为这个动作追加自己的 middleware。

李四说：“这就是聚合。看路由就知道 reset-password 是 users 资源的一部分。”

## collection action 的例子

后来图书模块要加热门书籍接口：

```txt
GET /books/hot
```

这个动作不属于某一本书，而是整个图书集合的一个视图。所以它应该是 collection action：

```go
sgin.ExtraAction{
    Method: "get",
    Path:   "hot",
    Detail: false,
    Handler: handlers.HotBooks,
}
```

导出图书也是类似：

```txt
GET /books/export
```

它面向集合，不需要 `:id`。

## 为什么需要 dispatcher

张三注册 `GET /books/hot` 时，遇到了一个 Gin 的底层事实：

```txt
GET /books/:id
GET /books/hot
```

这两个路由在 Gin 里不能总是自然共存。因为 `:id` 是通配段，`hot` 也占据同一个路径位置。

但从业务上看，这两个需求都合理：

- `GET /books/:id` 查询某本书。
- `GET /books/hot` 查询热门书籍。

不能因为框架路由树冲突，就禁止合理业务路径。

sgin 的做法是在 ViewSet 内部引入 dispatcher。对于可能和详情路由冲突的 collection action，请求会先进入统一入口。dispatcher 根据路径段判断：这一段是不是已声明的集合动作。如果是，就执行对应 action；如果不是，就按详情 ID 继续处理。

这让 `GET /books/hot` 和 `GET /books/:id` 可以在业务语义上共存。

## 为什么还要启动前冲突检查

张三问：“那如果我声明一个 collection action 叫 `123` 呢？”

李四说：“这就不应该放过。”

对于 `GET /books/123`，如果 ID 类型是 `uint`，它明显应该被理解为详情 ID，而不是集合动作。否则路由语义就会混乱。

所以 sgin 会在注册时做检查：collection action 的 path 不能被当前 ID 类型解析成合法 ID。比如对 `uint` ID 来说，`0`、`1`、`123` 这类路径会被拒绝。这样问题会在程序启动前暴露，而不是线上请求时才发现。

`0.5` 这种路径不能解析成 `uint`，就不会和 `/books/:id` 的整数 ID 语义冲突。

这点和 DRF 的函数名机制有相似目的。DRF 的 action 名来自 Python 函数名，函数名不可能是纯数字，也不能数字开头。sgin 没有函数名约束，所以用 path 的 ID 解析检查来保护路由语义。

## 动态路由权限看哪个 path

张三又想到权限系统：“如果请求实际进了 dispatcher，动态路由权限会不会看到 `/books/:id`，而不是 `/books/export`？”

李四说，这个坑已经处理过。

sgin 的 dispatcher 会把业务路径写入请求上下文。动态路由权限判断时优先使用业务 path，而不是 Gin 匹配到的通配路由。这样后台配置权限时，仍然配置直观的业务路径：

```txt
GET /books/export
POST /users/:id/reset-password
```

权限配置不应该泄漏内部 dispatcher 的实现细节。

## ExtraActions 不是什么都装

张三一旦学会 ExtraActions，就想把导入、支付、审批都塞进去。

李四提醒他：ExtraActions 适合资源内动作，不适合所有流程。

适合 ExtraActions 的动作通常满足：

- 它明显属于某个资源。
- 它的路径挂在该资源下更自然。
- 它可以复用该 ViewSet 的认证默认值、中间件和权限上下文，也可以对单个动作显式公开或强制登录。
- 它不是复杂的跨资源业务编排。

不适合的场景包括：

- 支付回调
- 外部 webhook
- 跨多个聚合根的审批流程
- 长任务批处理入口
- 和某个资源关系不强的系统操作

这些更适合普通 Gin handler + service。

## 这一章的判断题

李四让张三每次写动作前先问：

1. 这个动作属于某个具体对象，还是整个集合？
2. 如果属于具体对象，路径是否应该是 `/resources/:id/action`？
3. 如果属于集合，路径是否应该是 `/resources/action`？
4. action path 是否可能被 ID 类型解析成功？
5. 这个动作是否仍然是资源内动作，而不是跨资源流程？

答案清楚后，再决定 `Detail` 和 `Path`。

## 张三的笔记

张三在用户模块旁边写下：

资源内动作不要散落。用 `ExtraActions` 聚合到 ViewSet 下。`Detail` 表示是否绑定具体对象。collection action 要避开可解析成 ID 的路径。dispatcher 是为了让 `/books/export` 和 `/books/:id` 这类合理需求同时成立。
