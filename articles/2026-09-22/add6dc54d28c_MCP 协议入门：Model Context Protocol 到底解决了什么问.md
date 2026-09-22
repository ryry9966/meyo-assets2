---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38468
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

Agent 要干活，就得能碰到外部世界：查数据库、调内部 API、读文件、发消息。在 MCP 出现之前，这件事没有统一做法——每个 Agent 框架自定义一套 function calling 格式，每个工具提供方再为每个框架各写一次适配。3 个 Agent、8 个工具，理论上要维护 24 份胶水代码。MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，把"模型怎么发现工具、怎么调用工具、怎么读数据"标准化成一套基于 JSON-RPC 2.0 的接口。

## 它到底解决了什么

本质是把 **M×N 变成 M+N**：工具方只实现一次 MCP Server，Agent 侧（Host/Client）实现一次接入，双方按协议握手即可互通。协议里三个核心原语，边界要分清：

- **Tools**：模型可主动调用的动作，如"查询订单"；
- **Resources**：应用侧控制的只读数据，如配置、文档片段；
- **Prompts**：用户侧触发的模板。

传输层常见两种：stdio（本地子进程，最简单）和 Streamable HTTP（远程共享服务）。

## 动手：最小可跑通路径

1. 用官方 SDK（Python 或 TypeScript）起一个 stdio Server，注册一个工具，写清 name、description、inputSchema（JSON Schema）。
2. 在 Agent 宿主（比如 OpenClaw 的 MCP 接入配置）里挂上这个 server，重启会话。
3. 先跑 `tools/list`，确认 schema 被正确拉取；再手动触发一次 `tools/call`，观察结构化返回。
4. 本地跑通后，再考虑换 Streamable HTTP 部署成常驻服务。

先跑通一条链路再扩工具数量，别一上来铺十个工具。

## 踩坑记录

- **stdout 污染**：stdio 模式下 server 往 stdout 打日志，会破坏 JSON-RPC 消息流，表现为 client 解析失败。日志一律走 stderr。
- **description 写给人看，不是写给模型看**：模型靠工具名和描述决定调不调、怎么调。描述含糊，就会出现"明明有对的工具却调了别的"，或参数乱填。
- **工具太多**：几十个工具全量注入上下文，token 成本高且选择准确率下降。按场景拆 server，按需启用。
- **参数校验失败没有回路**：模型填错参数时，应把校验错误以可读文本返回给模型，让它有机会自我修正，而不是抛异常就结束。
- **安全别省**：工具结果是潜在注入入口，高危操作（删数据、外发请求）务必加确认机制或最小权限。

## 可复用建议

- 一个 server 只管一个领域，单 server 工具数控制在个位数。
- 工具设计尽量幂等、错误信息结构化，方便 Agent 重试。
- 排障时把 JSON-RPC 报文开到 debug 级别，大多数"没调工具"的问题都能在报文里找到原因。
- 先找社区现成 server，确认不满足再自己写。

## 总结

MCP 不提升模型智力，它降低的是**集成成本**：把"每次接工具都写胶水"变成"按标准实现一次，处处可插"。对做 Agent 自动化的人来说，它值得当作基础设施层认真对待——工具描述写好、边界划清、权限收紧，比追新功能更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/4dde336adbdb929b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/1e2f423509d7cd5b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/f72aeaf9e2ca4f4f.png)

