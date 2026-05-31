# 第 1 章：张三手写 CRUD 写到怀疑人生

张三入职第一天，李四让他做一个图书接口。

需求很普通：列表、详情、创建、更新、删除。

张三打开编辑器，准备写五个 handler、五段 GORM 查询、五套错误处理。写到删除接口时，他发现自己已经复制了很多格式相同的代码。

李四看了一眼，说：“这是普通资源 CRUD，不用从 handler 写起。”

## 张三的低效写法

张三把每个 HTTP 方法都当成独立接口处理：

- `GET /books` 手写查询
- `GET /books/:id` 手写按 ID 查找
- `POST /books` 手写绑定和创建
- `PUT /books/:id` 手写更新
- `DELETE /books/:id` 手写删除

这样当然能跑，但每个模型都要重复一遍。

## 李四的提醒

sgin 的 `ModelViewSet` 就是给这种场景准备的。

如果模型是普通数据库表，优先使用默认 GORM Repository。业务项目只需要注册模型和路由，标准 CRUD 就能自动获得。

## sgin 推荐写法

核心步骤：

1. 定义 GORM 模型。
2. 用 `app.InitTable(&models.Book{})` 初始化业务表。
3. 注册 `ModelViewSet[models.Book, uint]`。
4. 需要隐藏字段或调整响应时，再补 Serializer。

`ModelViewSet` 默认提供：

```txt
GET     /books
POST    /books
GET     /books/:id
PUT     /books/:id
PATCH   /books/:id
DELETE  /books/:id
```

## 这一章记住什么

普通资源 CRUD 不要先写 handler。先判断它是不是单表资源，如果是，优先用 `ModelViewSet` 和默认 GORM Repository。
