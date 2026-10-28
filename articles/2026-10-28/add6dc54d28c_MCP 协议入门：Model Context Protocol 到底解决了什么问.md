---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42826
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景：工具越多，胶水代码越多

Agent 的价值在于能"动手"：读文件、查数据库、操作浏览器、调内部 API。问题在于，过去每个 Agent 框架都有自己的工具定义格式，每个数据源都要单独写适配。3 个 Agent 配 8 个工具，理论上要维护 24 份互不通用的胶水代码。

MCP（Model Context Protocol）是 2024 年底由 Anthropic 开源的一套协议，目标就是把 N×M 问题压成 N+M：工具方实现一次 MCP Server，所有支持 MCP 的 Agent 都能用；Agent 侧实现一次 MCP Client，所有 MCP Server 都能接。

## 它具体定义了什么

MCP 基于 JSON-RPC 2.0，核心是三种能力：

- **Tools**：模型可主动调用的函数，最常用；
- **Resources**：可读入上下文的数据，比如文件、表结构；
- **Prompts**：预置提示词模板，社区用得少，知道即可。

传输层分两种：本地进程走 stdio，远程服务走 Streamable HTTP。个人自部署场景，stdio 是默认选择。

在 OpenClaw 里接入就是一个 `mcpServers` 配置段：写清启动命令、参数、环境变量，重启后 Agent 的工具列表里就会出现对应条目。

## 上手步骤

1. **先用现成的**。filesystem、fetch、sqlite、playwright 这类官方/社区 Server 已覆盖大部分本地自动化需求，别急着造轮子。
2. **最小配置跑通**。只挂一个 filesystem Server，限定到单一工作目录，验证工具能被发现、能被调用。
3. **看日志确认链路**。观察"模型选了哪个工具、传了什么参数、返回了什么"，这一步比跑通本身更重要。
4. **再写一个自己的**。用 Python 或 TypeScript SDK，几十行就能暴露一个查询内部系统的工具，写完对协议的理解会上一个台阶。

## 踩坑点

- **stdout 污染**：stdio 模式下 Server 的 stdout 只能跑协议消息，任何 print/调试输出都会让会话直接挂掉。日志一律走 stderr。
- **相对路径与 cwd**：Server 启动后的工作目录未必是你以为的目录，配置里一律写绝对路径；Windows 下注意中文路径编码。
- **工具描述质量决定调用质量**：description 是写给模型看的接口文档，含糊描述会导致该调不调、不该调乱调。
- **工具数量膨胀**：挂十几个 Server、几十个工具后，选择准确率明显下降。按任务场景动态启用，比全量挂载靠谱。
- **权限边界**：MCP Server 以你的用户身份运行，挂载整个家目录等于把所有文件交给模型上下文。最小权限目录 + 敏感操作二次确认是底线。
- **注入风险**：工具返回内容会进入上下文，网页抓取类 Server 尤其要警惕结果里夹带的指令。

## 可复用建议

- 一个 Server 只管一个领域（文件、数据库、浏览器分开），不要做万能 Server；
- 密钥走环境变量，别写在命令行参数里（进程列表可见）；
- 长任务考虑异步返回 + 轮询工具，避免阻塞会话超时；
- 工具粒度对齐"Agent 的一个动作"，而不是封装完整工作流——流程编排留给 Agent 层。

## 总结

MCP 解决的是"AI 应用与外部工具之间没有标准接口"的问题，把集成成本从 N×M 降到 N+M。它不解决权限治理、任务编排和安全审计——这些仍是部署者的责任。对 OpenClaw 用户的务实路径是：先用现成 Server 跑通链路，再用一个自写 Server 验证理解，最后按最小权限原则收口。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/2a1f57e77dd13989.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/ab5a51350f02073a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/76702e309fd0737b.png)

