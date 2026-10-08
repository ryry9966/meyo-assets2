---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 40874
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

过去一年做 Agent 自动化的人大概都有类似体会：模型能力够了，卡住的反而是"接线"。想让 Agent 读本地日志、查数据库、提 PR，每种数据源都要为特定框架写一遍胶水代码，换个框架基本作废。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标就是把这层接线标准化。可以粗略理解为 AI 应用领域的 USB-C：工具和数据源只要做一个 MCP Server，宿主（Agent、IDE、桌面客户端）只要实现 MCP Client，双方即插即用。

## 它到底解决什么问题

核心是 **M×N 集成问题**：M 个应用要接 N 个工具，传统做法要写 M×N 份适配代码。MCP 把它压成 M+N——每个工具写一个 Server，每个应用实现一次 Client。

协议基于 JSON-RPC 2.0，定义了三类能力：

- **Tools**：模型可主动调用的函数（对应 function calling）
- **Resources**：可读取的上下文数据（文件、表结构等）
- **Prompts**：预置的提示词模板

同样要说清楚它**不**解决什么：MCP 不负责编排、不提升模型智力、不替代权限治理。它只是一根标准化的管道。工具描述写得差、返回值里混入注入内容，模型照样犯错——这些责任仍在应用层。

## 跑通一条最小链路

1. **选场景**：从"给 Agent 加一个只读能力"开始，比如查内部 API 或读 SQLite。
2. **选 SDK**：官方有 Python（FastMCP）和 TypeScript SDK，几十行能起一个 Server。
3. **定义工具**：每个工具写清名称、参数 JSON Schema、一句话描述。
4. **本地调试**：用官方 MCP Inspector 连上 Server，手动调用看返回。
5. **接入宿主**：本地用 stdio 最简单；远程部署用 Streamable HTTP，注意会话管理。

以 FastMCP 为例，一个可用的 Server 核心就是 `@mcp.tool()` 装饰的函数加类型注解，协议细节 SDK 都包了。

## 踩坑点

- **工具描述就是提示词**：模型完全靠 name + description 选工具，描述含糊调用就乱。宁可啰嗦也要写清适用边界。
- **工具数量失控**：一个 Server 挂 30 个工具，上下文膨胀、选择准确率下降。按领域拆分 Server，保持单个工具集精简。
- **stdio 静默挂掉**：Server 崩溃在宿主端往往只剩一个笼统报错。先把 stderr 日志打出来再排查；Windows 下注意要写 `npx.cmd`。
- **安全是真实风险**：第三方 Server 拿着你的文件系统和凭证跑，工具返回值可能携带提示注入。生产环境只跑审计过的 Server，破坏性操作必须加人工确认。
- **大结果不截断**：一次返回几万 token 的查询结果会直接打爆上下文，Server 端要做分页和摘要。

## 可复用建议

- 工具粒度对齐领域动词：`create_issue` 好过 `call_api`，但也别拆到每个字段一个工具。
- 返回结构化、紧凑的结果，带上可读的错误信息，别只返回布尔值。
- 每个 Server 独立进程、独立权限，方便单独降级和替换。
- 记录每次工具调用的入参出参——排查问题基本靠这个。

## 总结

MCP 解决的是集成标准化，不是智能。如果 Agent 只接一个工具，手写 function calling 更省事；当集成对象超过两三个，或者想复用社区现成 Server 时，MCP 的投入才开始划算。它不性感，但属于那种"早接早省事"的基础设施，值得在下一个自动化项目里花半天时间跑通一遍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/b4e8b2083bc68af6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/bd290bb78b7d1121.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/46963f0babc0a205.png)

