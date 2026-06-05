# 第 3 章：张三为了一个列表接口写了一堆 handler

第三天上午，张三接到一个车辆列表需求。

这次需求更小：只有一个 URL，`GET /cars`。前端只要展示车辆列表，可以分页，可以按车牌号搜索，可以按状态过滤。没有详情，没有创建，没有更新，也没有删除。

张三吸取前两天经验，知道这不是完整资源 CRUD，也不是只读资源详情。他想了想，还是准备写一个普通 Gin handler：解析查询参数，调用 GORM，处理分页，返回统一响应。

李四路过时看到他又在写分页逻辑，问：“你只是少了几个路由，不是少了列表能力。为什么不用 APIView？”

## 单 URL 不等于从零开始

张三把问题想岔了。

他以为 `ModelViewSet` 是给一组 RESTful 路由用的；既然现在只有一个 URL，就只能回到普通 handler。

李四说：“这是两个维度。”

第一个维度是路由数量：一个 URL，还是一组资源 URL。

第二个维度是数据链路：是否仍然需要默认 Repository、Serializer、分页、过滤、认证、middleware。

车辆列表只有一个 URL，但它依然是一个普通数据库列表。它仍然需要分页、搜索、过滤和统一响应。把这些重新手写一遍，还是重复劳动。

## APIView 的定位

sgin 的 `APIView` 可以理解成单路由形态的 ViewSet 能力。

它适合这种场景：

- 只有一个 URL。
- 只有一个 HTTP method。
- 接口仍围绕某个模型或数据列表。
- 仍然想复用默认 Repository。
- 仍然想复用分页、过滤、Serializer、认证和 middleware。

张三的 `GET /cars` 正好符合。

它不是为了替代所有 Gin handler。它只是把“单个资源型接口”的重复链路收拢起来。

## 张三的 APIView 版本

张三把原来的 handler 草稿收起来，改成注册一个 APIView。

```go
app.Register(&sgin.APIView[Car, uint]{
    Method: "get",
    Path:   "/cars",
    Model:  &Car{},
})
```

如果没有提供自定义 handler，sgin 会根据 method 和 path 形态推断默认动作。对于 `GET /cars` 这种集合路径，它会走列表逻辑。

于是张三保留了单 URL 的简洁，又没有丢掉默认列表能力。

## APIView 和 ModelViewSet 的区别

李四让张三把两者放在一起看。

`ModelViewSet` 表达的是一个完整资源：

```txt
/books
/books/:id
```

它自然包含列表、创建、详情、更新、删除等 RESTful 动作。

`APIView` 表达的是一个具体入口：

```txt
/cars
```

它只注册你配置的 method 和 path。它不试图从一个 URL 推导出整个资源集合。

所以选择时不要问“哪个更高级”，而要问“这个需求到底是一组资源接口，还是一个单独入口”。

## APIView 什么时候很舒服

李四列了几个适合 APIView 的例子：

- 只有列表，没有详情的资源入口。
- 只有详情，没有列表的公开查询入口。
- 某个模型的轻量读取接口。
- 临时后台页面需要的单个数据接口。
- 不想暴露完整 CRUD，但仍想复用默认数据能力。

这些需求如果用普通 handler，代码会显得重复；如果用完整 ViewSet，又会显得路由过多。APIView 正好在中间。

## APIView 什么时候不该用

张三很快又产生了新想法：“那我把健康检查、webhook、支付回调也都用 APIView？”

李四立刻否定。

这些接口不适合 APIView：

- `/ping`
- `/health`
- 第三方 webhook
- 支付回调
- 审批动作
- 导入任务触发
- 多模型统计大屏
- 外部服务编排

原因很简单：它们的核心不是“围绕一个模型复用默认数据链路”，而是业务流程或系统信号。普通 Gin handler 更直接。

APIView 不是“所有单 URL 的统一容器”。它只适合那些仍然带有资源数据访问特征的单 URL。

## 自定义 handler 的位置

APIView 也可以配置自定义 handler。张三问：“那我什么时候要配 handler？”

李四说，如果默认推断动作不够表达业务，或者你需要在单 URL 上做一些特殊响应，可以配 handler。但要注意，一旦你完全接管 handler，就要自己负责业务流程。

换句话说，APIView 的价值不是“换一种方式注册 Gin 路由”，而是“在单 URL 下复用 sgin 默认链路”。如果你完全不需要这些链路，普通 Gin 路由更清楚。

## 和权限组合

车辆列表后来要求只有运维组可见。张三这次没有进 handler 写判断，而是在 APIView 上组合 middleware。

认证、加载访问控制、用户组判断，仍然属于请求链。默认配置下 APIView 也会先经过登录认证；如果这个单 URL 是公开查询入口，要在注册处显式允许匿名：

```txt
默认认证或 Auth
LoadAccess
RequireAnyGroup
默认列表逻辑
```

这和 ViewSet 的思路一致。APIView 虽然只有一个路由，但不应该放弃统一的权限组合方式。

## 这一章的判断题

李四让张三以后遇到单 URL 需求时，先问：

1. 这个 URL 是否围绕一个模型或资源列表？
2. 是否还需要分页、过滤、搜索或排序？
3. 是否想复用默认 Repository 或 Serializer？
4. 是否只有一个 method/path，而不是完整 RESTful 资源？

如果答案是肯定的，考虑 APIView。

如果它只是健康检查、回调、流程触发或跨模型编排，用普通 Gin handler。

## 张三的笔记

张三写下：

单 URL 不代表必须手写一切。只要它仍然是资源型数据入口，就可以用 `APIView` 复用默认能力；如果它是流程或系统信号，就回到 Gin handler。
