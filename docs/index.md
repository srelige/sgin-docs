# sgin Story Guide

张三刚入职，负责给一个内部系统写 API。李四在团队里用了很久 sgin，平时话不多，但每次 code review 都能指出张三哪里写得太累。

这套文档不是 API 参考，而是实践教程。每一章都用一个小故事说明：张三先用低效方式实现，李四再指出 sgin 已有的能力，以及什么时候不该用框架能力硬套。

## 适合谁

- 刚开始使用 sgin 的 Go 开发者
- 熟悉 Gin，但想少写重复 CRUD 的开发者
- 用过 DRF，想理解 sgin 取舍的人
- 不确定该用 ViewSet、APIView 还是普通 Gin handler 的人

## 阅读方式

按顺序读最顺。前几章讲资源接口，后几章讲权限、错误语义、分页和认证扩展。

| 章节 | 主题 | 你会学到 |
| --- | --- | --- |
| 01 | 手写 CRUD 写到怀疑人生 | `ModelViewSet`、默认 GORM Repository、`InitTable` |
| 02 | 复制只读接口 | `ReadOnlyModelViewSet` |
| 03 | 单 URL 列表接口 | `APIView` 的使用边界 |
| 04 | reset-password 放哪 | `ExtraActions`、detail/collection action |
| 05 | 不是所有接口都进 ViewSet | Gin handler + service 的边界 |
| 06 | 上传别塞进默认 CRUD | multipart 上传和 `rest.static_dir` |
| 07 | 别到处写 if admin | `JWTAuth`、`LoadAccess`、动态路由权限 |
| 08 | 前端分不清 401 和 403 | 稳定错误码和认证/权限边界 |
| 09 | 分页过滤别手搓 | page/page_size、搜索、排序、字段过滤 |
| 10 | API Key 不需要 AuthRegistry | Gin middleware 认证扩展 |

## 本站和 README 的关系

README 负责准确说明功能和配置。本教程负责讲使用场景和工程取舍。

如果你只想快速查字段，读 README。如果你想知道“这个接口到底该怎么写”，从第 1 章开始读。
