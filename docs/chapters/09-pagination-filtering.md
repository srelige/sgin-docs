# 第 9 章：分页过滤别手搓

张三写列表接口时，手动解析 `page`、`page_size`、`search`、`ordering`，还给每个字段写 if。

李四说：“普通列表的分页过滤，ViewSet 已经有默认链路。”

## 张三的低效写法

每个列表接口都重复：

- 解析页码
- 计算 offset
- 拼搜索条件
- 拼排序字段
- 校验过滤字段

字段一多，很容易出错。

## 李四的提醒

sgin 默认支持：

```txt
page
page_size
search
ordering
字段过滤
操作符过滤
```

分页由 `rest.pagination` 控制。默认关闭，开启后返回 `total/page/page_size/results`。

## 过滤字段要白名单

业务字段过滤必须通过 `FilterFields` 声明。排序字段也必须通过 `OrderingFields` 声明。

这样做是为了避免用户随便把查询参数映射到数据库字段。

## 这一章记住什么

后台系统常规列表用 `page/page_size` 足够。需要特殊查询时再自定义 Repository，不要每个 handler 手搓分页过滤。
