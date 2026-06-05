# 第 1 章：张三手写 CRUD 写到怀疑人生

周一上午九点半，张三领到入职后的第一个需求：给内部图书资产表做一组接口。

产品同事说得很轻松：“就是图书列表、详情、新增、编辑、删除。后台自己用，不复杂。”

张三点点头。Gin 他会，GORM 他也会。需求看起来没有任何陷阱。他打开编辑器，新建 `book_handler.go`，准备一口气写完。

第一个接口是列表。张三写查询参数绑定，读 `page` 和 `page_size`，算 offset，查数据库，返回 JSON。第二个接口是详情，他从 URL 里取 `id`，转成整数，查不到就返回 404。第三个接口是创建，绑定 JSON，校验字段，调用 `db.Create`。第四个接口是更新，先查旧记录，再覆盖字段。第五个接口是删除，先查存在性，再删除。

午饭前，他把五个接口都写出来了。看上去能跑，但李四 review 时只问了一句：

“你觉得这五段代码，明天换成资产、部门、供应商，是不是还要再写一遍？”

张三沉默了一会儿。他刚刚写完的东西，确实每段都很熟悉，也每段都很像废话。

## 张三第一版为什么会变重

张三的问题不是写错了，而是把“普通资源 CRUD”当成了“业务流程”。

普通资源有非常稳定的形态：

```txt
GET     /books        列表
POST    /books        创建
GET     /books/:id    详情
PUT     /books/:id    全量更新
PATCH   /books/:id    局部更新
DELETE  /books/:id    删除
```

这类接口的差异主要来自模型、序列化、权限和过滤字段，而不是 handler 流程。张三从 handler 写起，就会在每个模型上重复这些事情：

- 解析 ID
- 绑定请求体
- 调用数据库
- 处理不存在
- 处理分页
- 包装统一响应
- 做基础错误转换
- 注册一组 RESTful 路由

第一次重复时不明显。等系统里有十几个资源，重复就会变成维护成本。

李四在 review 里给张三画了一条线：

“如果一个接口的主语是一个数据库资源，动词又刚好是增删改查，那你先别写 handler。先问它是不是一个 ViewSet。”

## sgin 眼里的普通资源

sgin 的 `ModelViewSet` 就是为普通资源准备的。

它不要求你为每个模型写 Repository。只要你传入 GORM 模型，且当前 App 有默认数据库连接，sgin 会使用默认 GORM Repository 完成基础 CRUD。

在张三的图书例子里，真正需要业务项目表达的是三件事：

1. 这个资源的模型是什么。
2. 这个资源挂在哪个路径下。
3. 这个资源有哪些额外配置，比如认证、权限、过滤、序列化。

剩下的标准 CRUD 流程不应该散落在每个 handler 里。

## 李四让张三重写

李四没有让张三把所有代码删掉。他让张三先保留模型，然后重新审视路由注册。

张三把图书表定义成一个普通 GORM 模型：

```go
type Book struct {
    ID     uint   `json:"id" gorm:"primaryKey"`
    Title  string `json:"title"`
    Author string `json:"author"`
    Status string `json:"status"`
}
```

然后在启动时初始化业务表：

```go
app.InitTable(&Book{})
```

最后注册资源：

```go
app.Register(&sgin.ModelViewSet[Book, uint]{
    BasePath: "/books",
    Serializer: sgin.ModelSerializer[Book]{
        ReadFields:  []string{"id", "title", "author", "status"},
        WriteFields: []string{"title", "author", "status"},
    },
})
```

张三看着这几行代码，有点不放心：“那列表、详情、创建、更新、删除都在哪里？”

李四说：“它们不是没了，是被收拢到一个稳定模式里了。”

## 默认 GORM Repository 做了什么

当 `ModelViewSet` 没有显式传入 `Repository` 时，sgin 会尝试使用默认 GORM Repository。

它负责基础数据访问：

- `List`：按查询条件列出数据
- `Find`：按 ID 查找详情
- `Create`：创建记录
- `Update`：更新记录
- `Delete`：删除记录
- `Count`：在分页开启时统计总数

张三以前手写的很多代码，并不是业务规则，而是每个资源都要重复的数据库动作。默认 Repository 的价值就是把这些重复动作变成框架能力。

如果某个资源不是 GORM 表，或者查询逻辑非常特殊，再传自定义 Repository。不要为了显得“可控”，在普通表上先手写 Repository。

## `InitTable` 不是业务迁移系统

张三看到 `app.InitTable(&Book{})` 后，又问：“那我是不是可以把所有数据库迁移都交给它？”

李四摇头。

`InitTable` 的目标很朴素：在简单后台系统里，帮你初始化业务模型表。表不存在时创建，表已存在时跳过。它适合快速启动、演示项目、内部工具、普通后台资源。

它不是完整迁移系统。复杂生产系统如果需要严格 schema migration、回滚、灰度字段、数据迁移脚本，仍然应该使用专门迁移工具。

sgin 不试图把所有工程问题都吞进去。

## Serializer 的位置

张三接着问：“如果我不想把某些字段返回给前端怎么办？”

李四说：“这才是你应该扩展的地方。”

普通 CRUD 流程可以交给 `ModelViewSet`，但响应字段、输入字段、列表和详情的形态，可能会因业务不同而变化。sgin 要求默认数据链路显式配置 Serializer，就是让你在不重写 CRUD 流程的前提下控制输入输出。

李四特意提醒张三：“不要让模型悄悄变成完整请求体和完整响应体。你要么用 `ModelSerializer` 写清楚读字段和写字段，要么显式选择 `FullModelSerializer`，表示你确实接受全量模型读写。”

写入侧也不是静默忽略。假设 `WriteFields` 里只有 `title`、`author`、`status`，请求里传了 `is_admin`、`owner_id` 或其他未声明字段，sgin 会直接返回 400。这样张三马上能知道有人在尝试写非白名单字段，而不是等数据被悄悄污染后再排查。

也就是说，不要为了隐藏字段去复制默认 handler。先考虑 Serializer。

这个原则很重要：

- 流程重复，交给 ViewSet。
- 字段形态不同，交给 Serializer。
- 存储逻辑不同，交给 Repository。
- 业务动作不同，交给 ExtraActions 或普通 handler。

## 权限和中间件也不该写进 CRUD

张三的第一版代码里，每个 handler 都准备以后补一段权限判断。

李四让他先停住：“权限是请求链上的事情，不要埋到每个 CRUD 函数里。”

`ModelViewSet` 支持默认登录认证和 middleware。默认配置下资源接口需要登录；如果某个列表或详情要公开，可以显式配置匿名访问。如果项目把全局认证改成默认公开，也可以用 `Auth` 把指定接口重新保护起来。如果还需要加载用户访问控制，再串 `LoadAccess()`、`RequireAnyGroup()` 或 `RequireRoutePermission()`。

这不是本章重点，但张三提前记住了一件事：handler 里越少混入通用横切逻辑，后续越不容易乱。

## 这一章的判断题

李四最后给张三留了一个小练习。看到一个新需求时，先回答这几个问题：

1. 它是不是围绕一个资源展开？
2. 它是不是标准增删改查？
3. 它是不是普通数据库模型？
4. 它的差异是不是主要在字段、权限、过滤上？

如果答案大多是肯定的，先用 `ModelViewSet`。

如果你一开始就想写五个 handler，通常要停下来想想：你是在写业务，还是在重复框架已经能做的事情？

## 张三的笔记

那天下午，张三把原来的五个 handler 删掉，只留下模型、表初始化和 ViewSet 注册。

他在笔记里写了一句话：

普通资源 CRUD 不要从 handler 开始。先让 `ModelViewSet` 接住稳定流程，再把真正不同的地方放到 Serializer、Repository、权限或额外动作里。
