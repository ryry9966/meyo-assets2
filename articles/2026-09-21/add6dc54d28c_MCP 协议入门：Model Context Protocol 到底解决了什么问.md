---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38287
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

过去两年写 Agent 自动化的人基本都绕不开一个尴尬：模型本身能力不差，但它被困在对话框里，碰不到你的数据库、内部 API 和本地文件。要让 Agent 真正干活，就得给它接工具。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开放的一套协议，目标是把"模型怎么接工具"这件事标准化。一年多下来，它已经成了事实标准，主流客户端和大量社区 Server 都在支持。

## 它到底解决了什么问题

在 MCP 之前，接工具是一个 M×N 问题：M 个 Agent 框架 × N 个数据源，每个组合都要写一遍胶水代码。你的消息机器人、CLI 助手、CI 工具各自实现各自的调用格式，几乎无法复用。

MCP 把 M×N 变成 M+N：Agent 框架只需实现一次 MCP Client，每个数据源只需实现一次 MCP Server，中间用统一协议对话。这就是它的核心价值——不是让模型更聪明，而是让整个生态的接口长一个样。

## 协议结构：三分钟版

MCP 采用 Host–Client–Server 架构：

- **Host**：运行模型的宿主应用，比如你的 Agent 运行时
- **Client**：Host 内的连接器，与单个 Server 一对一通信
- **Server**：暴露能力的轻量进程，可以是本地脚本，也可以是远程服务

Server 对外提供三类原语：

- **Tools**：模型可主动调用的动作，如"查订单""发消息"
- **Resources**：可读取的上下文数据，如文件内容、表结构
- **Prompts**：预置的提示模板

传输层常见两种：`stdio`（本地子进程，适合个人自动化）和 Streamable HTTP（远程部署，适合团队共享）。

## 上手步骤

以接一个内部 REST API 为例：

1. 选 SDK：官方提供 Python（FastMCP）和 TypeScript SDK，几十行就能起一个 Server
2. 定义工具：把 API 的每个操作包成一个 tool，写清 name、description 和参数 schema
3. 本地跑通：用 stdio 模式启动，在支持 MCP 的客户端里手动调用一次，确认返回结构正常
4. 接入 Agent：在 OpenClaw 类运行时里注册这个 Server，观察模型是否正确选到工具
5. 看日志：重点核对模型传入的参数和你的预期差多少

## 踩坑点

- **description 含糊，模型就选错或不选工具**。工具名和描述本质是写给模型看的文档，按 API 文档的标准来写。
- **工具一多模型就懵**。单个 Server 控制在十来个 tool 以内，超了就按业务域拆分。
- **stdio 进程的生命周期**：Server 随 Host 退出而退出，长任务要处理中断；调试信息别打到 stdout，会污染协议帧。
- **安全**：第三方 Server 能以你的权限执行动作，接入前审一遍代码；生产环境最小权限，高危操作加人工确认。
- **版本兼容**：协议仍在演进，Client 和 Server 版本差太多会出现 capability 协商失败。

## 可复用建议

- 把内部系统逐个包成 MCP Server，比每个 Agent 重复造轮子划算得多
- 维护一个小型评测集，验证"模型选工具的准确率"，改 description 前后各跑一遍
- 工具返回值尽量结构化且短，长文本让模型按需分页拉取

## 总结

MCP 解决的不是模型能力问题，而是集成成本问题。它把混乱的胶水层收敛成一个标准接口，让"接工具"从一次性工程变成可积累的资产。对做 Agent 自动化的团队来说，越早把内部能力 MCP 化，后续的组合成本就越低。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/40bbdb2b0745f6ab.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/4568856e963e1838.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/95319ac7d5448149.png)

