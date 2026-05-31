# 第 6 章：上传别塞进默认 CRUD

张三要做文件上传。他想把文件直接塞进 `ModelViewSet.Create`。

李四马上拦住他：“默认 CRUD 面向 JSON，上传是 multipart，要单独设计。”

## 张三的低效写法

把上传混进默认创建接口：

- 一会儿处理 JSON
- 一会儿处理 multipart
- 文件校验和模型绑定混在一起
- 入库字段也不清晰

后续维护会很难。

## 李四的提醒

上传入口应该显式处理。

常见形态：

```txt
POST /files
POST /assets/:id/attachments
POST /imports/books
```

可以用普通 Gin handler，也可以在资源内用 `ExtraActions`。

## 上传 handler 应该做什么

上传流程通常包括：

- 读取 `FormFile`
- 校验大小
- 校验扩展名或 MIME
- 保存到 `rest.static_dir` 或对象存储
- 入库保存路径、对象 key、文件名、大小等元数据

sgin 不会自动保存文件，也不会自动注册静态文件服务。

## 这一章记住什么

JSON CRUD 和 multipart 上传是两类入口。不要让默认 `Create` 同时背两个职责。
