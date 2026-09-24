---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 38771
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

OpenClaw 的核心循环是「模型决策 + 工具执行」。模型本身只会生成文本，真正让它"动手"的是暴露给它的工具层。对接外部服务——工单系统、内部数据库、各类 SaaS API——几乎是每个 Agent 项目的必经之路。在 OpenClaw 里可选的路径有三条：原生 tool 调用、MCP 服务器、以及插件目录里的自研扩展。

## 问题

裸接 API 的常见翻车姿势：

- 模型靠猜填参数，字段名错一半；
- 返回 500KB 的原始 JSON，直接撑爆上下文；
- 写操作没有幂等保障，一次重试下了两单；
- 密钥写进 system prompt，日志里裸奔。

这些问题的共性是：把"API 原样"当成了"Agent 工具"，中间缺一层适配。

## 做法与步骤

1. **选接入方式。** 逻辑简单的 REST 服务用原生 tool；需要多工具聚合、跨项目复用的走 MCP server；带复杂状态或本地能力的写成插件。
2. **定义窄接口。** 不要把 API 全量暴露。一个工具只做一件事，JSON Schema 里写清参数类型、枚举值、必填项。`description` 里放"什么时候该用 / 不该用"的判断条件，比堆字段更有用。
3. **加一层薄适配。** 在 handler 里做三件事：过滤响应字段（只留模型需要的）、把错误归一化成 `{ok, error_code, hint}` 结构、设置 3~5 秒超时。
4. **密钥走环境变量。** 通过 OpenClaw 的 secret 机制注入进程环境，不出现在任何 prompt 或明文配置里。
5. **先灰度再放开。** 写操作加 `dry_run` 参数，跑一轮真实任务看 tool call 日志，确认行为后再放开。

一个最小 schema 示例：

```json
{
  "name": "create_ticket",
  "description": "创建工单。仅在用户明确要求提交时调用；查询请用 search_ticket。",
  "parameters": {
    "type": "object",
    "properties": {
      "title": { "type": "string" },
      "priority": { "enum": ["low", "mid", "high"] },
      "dry_run": { "type": "boolean", "default": true }
    },
    "required": ["title"]
  }
}
```

## 踩坑点

- description 写"调用工单 API"这种废话，结果是该调的时候不调、不该调的时候乱调。要写触发条件，不要写功能介绍。
- 重试必须带指数退避，否则触发限流时你的重试就是在 DDoS 自己。
- MCP server 进程挂了不会主动报错，只会表现为"模型突然不调工具了"。上线后加健康检查和心跳日志。
- 响应裁剪时不要把错误信息裁掉——模型需要错误内容才能自我纠正。

## 可复用建议

- **错误结构化**是投入产出比最高的一件事：模型拿到 `error_code + hint`，多数情况能自己改参数重试，不用人工介入。
- 给每次工具调用打 `trace_id`，与上游 API 的 request id 串起来，排障时间从小时级降到分钟级。
- 参数校验放在适配层做，prompt 只负责引导，不负责兜底。

## 总结

对接外部服务的本质，不是"把 API 塞给模型"，而是设计一层人、模型、上游服务三方都能读懂的窄接口。接口收窄、错误结构化、写操作可灰度——这三件事做到位，Agent 与 API 的握手才算真正握实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/9cf3bcb7743ca2bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/fd39c00378ca9414.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/1bac3f974529e396.png)

