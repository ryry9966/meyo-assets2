---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 38456
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

Agent 的上限往往不取决于模型多聪明，而取决于它能调用什么。在 OpenClaw 的架构里，agent 本体只负责推理与决策，真正"动手"的部分——查订单、发通知、写数据库——都要通过工具层完成。这层怎么对接外部 API，直接决定 agent 是"能用"还是"好坑"。

## 问题

直接把外部 API 包一层就暴露给 agent，通常会撞上几个现实：

- 鉴权、分页、限流这些约定是给人写的，不是给模型写的；
- 工具描述含糊时，模型会乱填参数、瞎猜字段；
- 一个接口返回 50KB 的原始 JSON 直接塞进上下文，token 烧得飞快；
- 网络抖一下，agent 可能对同一个非幂等接口连重试三次。

## 做法

我们的原则是：**API 不迁就 agent，adapter 迁就双方**。在 OpenClaw 里对接一个外部服务，大致四步：

1. **选层**。通用能力（搜索、文档查询）走 MCP server，方便跨项目复用；业务私有接口（内部工单、自有数据库）写成插件/本地工具，收敛在一处管理。
2. **收窄接口**。不要把 REST 的十几个 endpoint 全注册成工具。按 agent 实际任务切分，一个工具做一件事，命名用动词+宾语（`create_ticket`，而不是 `ticket_api_call`）。
3. **写好 schema 与描述**。description 是写给模型看的：什么时候该用、什么时候别用、参数什么格式。这段描述的价值不亚于代码本身。
4. **在 adapter 里处理工程问题**：鉴权走环境变量，密钥永不进上下文；超时与重试只作用于幂等操作；错误归一化，把 HTTP 500 翻译成模型能理解并据此决策的一句话，比如"额度不足，任务已暂停，请告知用户"。

一个简化的工具定义示意：

```json
{
  "name": "search_orders",
  "description": "按用户ID查最近订单。用户明确要求查订单时才调用；不要用于查物流。",
  "parameters": { "user_id": "string", "limit": "number, 默认5" },
  "timeout_ms": 8000
}
```

## 踩坑点

- **"工具越多越好"是错觉**。注册 30 个工具后，模型选择准确率明显下降，单 agent 10 个以内最稳。
- **直接返回原始 JSON**。应在 adapter 里裁剪字段、压平结构，只回传决策需要的部分。
- **重试不分幂等**。`GET` 可以重试，`POST` 支付/下单类操作必须带幂等键，否则会重复下单。
- **密钥泄漏**。日志和异常信息里都可能带 token，adapter 出口统一做脱敏。
- **接口漂移**。外部 API 改字段不会通知你，schema 校验要快速失败并告警，而不是让模型拿到 `undefined` 硬猜。

## 可复用建议

- 所有外部调用走同一层薄 adapter，鉴权、超时、日志、脱敏只写一遍；
- 每个工具加 dry-run 模式，上线前先跑一轮回归；
- 用 contract test 锁住外部 API 的关键字段，漂移时先炸测试、再炸 agent；
- 把"何时调用/何时不调用"写进工具 description，比在 system prompt 里堆规则有效得多。

## 总结

Agent 与外部服务的握手，本质是两种"语言"的翻译：API 面向确定性调用，agent 面向概率性推理。OpenClaw 的实践里，这层翻译做扎实了，模型能力才能兑现；做糊了，再强的模型也在猜接口。工具层，值得你花掉一半的集成时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/b74dd03a49ad764f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/40d53537aa59fc39.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/de31a8e785cfb457.png)

