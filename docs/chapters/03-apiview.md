# 第 3 章：张三为了一个列表接口写了一堆 handler

张三接到一个需求：只需要 `GET /cars`，返回车辆列表。

他又准备写 handler、查库、分页、过滤。李四问：“你只是要一个 URL，为什么不用 APIView？”

## 张三的低效写法

单个 URL 也从零开始写：

- 手写绑定查询参数
- 手写 Repository 调用
- 手写分页包装
- 手写统一响应

如果只是一个简单的列表接口，这些工作都重复了。

## 李四的提醒

`APIView` 是 `ModelViewSet` 的单路由子集。

它适合：

- 只有一个 URL
- 只有一个 HTTP method
- 仍然想复用默认 Repository、Serializer、分页、过滤

## sgin 推荐写法

例如只需要列表：

```txt
GET /cars
```

可以用 `APIView` 配置 method 和 path。没有自定义 handler 时，它会按 method/path 推断默认动作。

## 什么时候不用 APIView

不要把所有单 URL 都塞给 APIView。

不适合：

- `/ping`
- `/health`
- webhook 回调
- 支付、审批、导入这类流程接口

这些用普通 Gin handler 更直接。

## 这一章记住什么

单 URL 且想复用默认数据库能力时，用 `APIView`。不需要数据库能力时，用 Gin handler。
