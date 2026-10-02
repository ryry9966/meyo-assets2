---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 40131
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

写过 Agent 的人大概率经历过这种场景：想让模型读数据库、调内部 API、操作文件，于是给每个能力手写一套 function calling 的 schema 和调用逻辑。换一个框架，这些胶水代码全部重写。接 N 个模型、M 个工具，就要维护 N×M 份连接代码。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标是把「模型应用 ↔ 外部能力」这一层标准化。可以类比为 USB-C：Host（Agent 运行时）不关心外设内部实现，Server（能力提供方）也不用为每种设备做定制。

## 它到底解决什么问题

三个：

1. **集成爆炸**。没有 MCP 时，每个框架对每个工具都要写专用连接器。有了 MCP，工具方实现一次 Server，任何支持协议的 Host 都能直接用，复杂度从 N×M 降到 N+M。
2. **上下文割裂**。工具描述、参数 schema、错误格式散落各处，模型看到的上下文质量参差不齐。MCP 规定了统一的发现机制（`tools/list`）和调用格式（`tools/call`），工具描述成为一等公民。
3. **三类能力混用**。MCP 明确区分 Tools（模型调用、有副作用）、Resources（应用控制的数据读取）、Prompts（用户触发的模板）。很多问题恰恰源于把该做成 Resource 的数据做成了 Tool。

## 做法：跑通一条最小链路

以 OpenClaw 这类支持 MCP 的运行时为例：

1. **选传输方式**。本地进程用 stdio（子进程拉起，零部署），远程服务用 Streamable HTTP。个人工具链从 stdio 起步最省事。
2. **先复用再自写**。社区的 filesystem、sqlite、browser 等 Server 已覆盖大部分场景，先挂上跑通，再决定要不要自己写。
3. **自写用官方 SDK**（Python/TypeScript），核心就是声明工具名、JSON Schema 参数和一段描述。一个查询订单的工具，30 行以内能跑起来。
4. **用 Inspector 单测**。官方 MCP Inspector 可以脱离 Agent 单独验证 Server，确认 `tools/list` 返回的 schema 和描述符合预期。
5. **接入后观察**模型的工具选择行为，持续迭代描述文案。

## 踩坑点

- **描述就是提示词的一部分**。模型靠 description 选工具，「查询数据」这种描述在多工具场景下必然选错。要写到「按订单号查询某用户近 30 天已支付订单，返回 JSON」的精度。
- **stdio 下 stdout 是协议通道**。Server 里任何 print/log 走了 stdout 都会污染协议流，日志必须走 stderr。这是新手第一名。
- **工具数量失控**。挂 50 个工具，光 schema 就吃掉大量上下文，选择准确率也下降。按领域合并成少量粗粒度工具，比一堆细碎工具好用。
- **有副作用的工具要设计确认与幂等**。删除类操作至少要求显式确认参数，别把「删除全部」裸露出去。
- **别依赖跨会话状态**。MCP Server 默认按会话隔离，持久记忆要自己落到外部存储。

## 可复用建议

- 内部 API 按领域聚合成一个 Server，而不是一个接口一个 Server。
- 工具命名用「动词_宾语」：`query_order`、`create_ticket`。
- 每次工具调用的入参、出参、耗时全部落日志——排查「模型选错工具」时，这是唯一可靠证据。
- 读多写少的数据优先做成 Resource，动作类才做成 Tool。

## 总结

MCP 没有引入新魔法，它只是把 Agent 生态里最脏的胶水层标准化了。真实收益有三点：能力可复用、运行时可替换、工具设计变成可评审的工程问题。判断标准也很朴素——同一个工具你已经在第二个 Agent 里重写了第二次，就该把它包成 MCP Server 了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/b7218aa300b89d63.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/ff53c622324be2ad.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/13a7e495929ccdeb.png)

