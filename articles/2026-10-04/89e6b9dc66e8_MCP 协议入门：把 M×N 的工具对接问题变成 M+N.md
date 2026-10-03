---
title: MCP 协议入门：把 M×N 的工具对接问题变成 M+N
feedId: 40344
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

做 Agent 开发的同行大概率都经历过这个循环：给模型接一个新工具，就要写一段 function calling 的胶水代码——定义 schema、处理调用、拼接结果、处理鉴权。接两个工具还好，接到第十个你会发现，换一个宿主（换 Agent 框架或客户端），这套胶水还得重写一遍。每个客户端有自己的插件格式，每个工具方维护 N 份适配代码，这就是典型的 M×N 问题。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标只有一个：让「模型应用」和「工具/数据源」之间有一层标准接口。工具方实现一次 MCP Server，任何实现 MCP Client 的宿主都能用；反过来宿主也只需实现一次 Client。M×N 变 M+N。

## 协议本身提供了什么

MCP 基于 JSON-RPC 2.0，核心概念不多：

- **Tools**：模型可主动调用的能力，带输入 schema，可以理解为 function calling 的标准化版；
- **Resources**：宿主可读取的上下文数据，比如文件、数据库记录；
- **Prompts**：预置提示词模板，由用户显式触发；
- **传输层**：本地用 stdio，远程用 Streamable HTTP。

连接建立时有一次 initialize 握手，双方协商协议版本和各自支持的能力。就这些，没有魔法。

## 动手：最小可用链路

1. 用官方 SDK（Python 或 TypeScript）写一个 stdio server，注册两个工具，比如 `list_files` 和 `read_file`，把参数 schema 和工具描述写清楚；
2. 用官方 Inspector（`npx @modelcontextprotocol/inspector`）本地调试，确认工具能被枚举和调用；
3. 在宿主的配置文件里挂上这个 server（多数客户端就是一段 JSON，指明命令和参数）；
4. 在 Agent 里发起一次真实调用，观察工具描述如何进入上下文、调用结果如何回填。

走完这条链，你对 MCP 的理解会比读十篇科普扎实。

## 踩坑点

- **stdio server 往 stdout 打日志**。stdout 是协议通道，print 一行调试信息整个连接就挂了。日志一律走 stderr；
- **工具描述质量决定调用质量**。描述含糊，模型就会瞎传参数。把描述当成写给「一个很聪明但不了解你业务的新同事」的接口文档；
- **返回结果过大**。工具一把返回几万 token，上下文直接被吃掉。要做分页、截断、摘要；
- **工具数量失控**。挂几十个 server，几十个工具定义全塞进 prompt，成本和准确率一起崩，按需启用；
- **安全边界**。工具返回的内容会被模型当作上下文，天然存在注入面。写操作类工具要有确认机制，只读和可写分开。

## 可复用建议

- 工具粒度宁粗勿细：一个 `create_issue` 胜过五个原子操作，除非确实需要模型编排；
- 每次调用都留日志：参数、耗时、返回大小，排查问题时这是唯一的救命稻草；
- 接入顺序固定为：Inspector 手测 → 接入 Agent → 再谈自动化，不要跳步。

## 总结

MCP 解决的不是「模型不够聪明」，而是工程层面的重复对接。它把工具生态从点对点胶水中解放出来，代价是多一层协议要学、多一类故障面要防。对 OpenClaw 这类强调插件与自动化的场景，理解 Tools/Resources 的边界、管住返回体积和工具数量，比急着多接十个 server 更有价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/eafb110beef0711e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/3ca2848594b3b3b1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/fcbe4928a403c85c.png)

