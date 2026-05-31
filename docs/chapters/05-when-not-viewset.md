# 第 5 章：不是所有接口都该进 ViewSet

张三越来越熟悉 ViewSet，于是他想把支付、审批、导入、批处理都塞进 ViewSet。

李四说：“ViewSet 是资源接口工具，不是所有业务流程的容器。”

## 张三的低效写法

他把复杂流程写进 ViewSet handler：

- 请求体和模型完全不同
- 一个请求写三张表
- 还要调用外部支付接口
- 最后还要写审计日志

代码能跑，但边界很乱。

## 李四的提醒

遇到流程型接口，优先用 Gin handler + service。

适合普通 handler 的场景：

- 支付
- 审批
- 导入
- 批处理
- webhook
- 多模型事务
- 外部服务编排
- 长任务触发

## 推荐分层

handler 负责 HTTP：

- 解析请求
- 调用 service
- 返回响应

service 负责业务：

- 校验规则
- 事务编排
- 调外部服务
- 写审计

DAO/Repository 负责数据访问。

## 这一章记住什么

不要为了复用 ViewSet 扭曲业务边界。复杂流程用普通 Gin handler 更清楚。
