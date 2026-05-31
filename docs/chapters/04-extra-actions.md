# 第 4 章：reset-password 到底放哪

张三要给用户加一个重置密码接口。

他先写了一个普通路由：

```txt
POST /reset-user-password
```

李四看完说：“这个动作属于用户资源，应该跟 `/users` 聚合在一起。”

## 张三的低效写法

资源内动作散落在普通路由里：

```txt
POST /reset-user-password
POST /sync-asset
GET  /export-books
```

时间久了，路由会越来越不像一个资源 API。

## 李四的提醒

sgin 用 `ExtraActions` 表达资源内非 CRUD 动作。

detail action 属于某个对象：

```txt
POST /users/:id/reset-password
POST /assets/:id/sync
```

collection action 属于整个集合：

```txt
GET /books/export
GET /books/hot
```

## dispatcher 为什么存在

Gin 不能直接同时注册：

```txt
GET /books/:id
GET /books/export
```

因为 `:id` 是通配段。sgin 在 ViewSet 内部做 dispatcher：请求进入详情入口后，先判断这一段是不是集合动作；如果是，就执行集合动作；否则继续走详情查询。

## 这一章记住什么

资源内动作不要散落。用 `ExtraActions` 聚合到资源 ViewSet 下。跨资源流程才考虑普通 handler。
