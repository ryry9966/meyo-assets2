---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38265
source: 综合讨论
publishedAt: 2026-09-20
---

# 背景：接线比调 prompt 更耗时

做 Agent 开发一段时间后你会发现，最耗时间的往往不是 prompt 调优，而是"接线"：让模型读到你的数据库、调用内部 API、操作本地文件。每接一个工具，就要写一套 function calling 的 schema、一套鉴权、一套错误处理；换个 Agent 框架，这些胶水代码基本要重写一遍。

这是典型的 M×N 问题：M 个 Agent 框架 × N 个数据源，需要 M×N 份集成代码，而且互不通用。

# MCP 解决了什么

Model Context Protocol（MCP）是 Anthropic 在 2024 年底开源的协议，思路很直接：把 M×N 收敛成 M+N。工具方按协议实现一次 Server，Agent 侧实现一次 Client，双方通过标准消息通信。

几个关键点：

- 消息层基于 JSON-RPC 2.0，主流 transport 有两种：stdio（本地子进程）和 Streamable HTTP（远程服务）
- 架构分三层：Host（Agent 应用）、Client（Host 内的连接器）、Server（工具提供方）
- 对模型暴露三类原语：**tools**（可调用函数）、**resources**（可读取数据）、**prompts**（可复用提示模板）

说白了，MCP 没有发明新能力，它定义的是"Agent 怎么发现工具、怎么调用、怎么拿结果"的标准接口。真正的价值在生态：filesystem、git、Postgres、浏览器这些常用 Server 已经有人写好，装上就能用。

# 上手步骤

以最常见的需求——让 Agent 读写本地文件——为例：

1. **选 transport**。本地场景用 stdio，Server 作为子进程启动，最省事；跨机器或生产环境用 Streamable HTTP。
2. **配置 Host**。在支持 MCP 的客户端配置文件里声明 server：启动命令、参数、环境变量。
3. **验证握手**。启动后确认 Client 能拿到 Server 返回的工具列表和参数 schema。
4. **写自己的 Server（可选）**。官方有 Python / TypeScript SDK，一个 tool 就是"一个函数 + 一段 JSON Schema 描述"，几十行能跑通。

# 踩坑点

- **工具描述就是 prompt**。模型只靠 name / description / schema 决定调不调、怎么调。描述含糊，调用就含糊。这里投入产出比最高，别省。
- **工具别贪多**。工具一多，schema 撑爆上下文，模型选择准确率也会掉。按场景裁剪，单个 Server 十几个工具以内比较稳。
- **安全边界想清楚**。stdio 模式下 Server 跑在你本机，继承你的全部权限。装第三方 Server 前先读代码，至少搞清它能碰哪些路径和网络。
- **Spec 迭代快**。transport 方案半年内改过几轮，网上旧教程的 SSE 写法可能已废弃，以官方 spec 和 SDK 文档为准。
- **返回内容要克制**。别把整张查询结果怼给模型，做分页、截断、字段裁剪，token 和模型注意力都是成本。
- **长任务要设计**。同步等一个跑几分钟的工具容易撞超时，要么拆小，要么异步化加轮询。

# 可复用建议

- 先用现成 Server 验证场景，再决定自己写。很多需求 filesystem + git + 一个 fetch 就够了。
- 工具粒度按"模型的一次决策"来切，而不是按你的代码结构切；一个工具只做一件事。
- 把每个 MCP Server 当独立服务管理：独立日志、独立重启、权限最小化。
- 鉴权和限流放在 Server 层做，不要指望模型"记得"不越权。

# 总结

MCP 本质上是一层薄薄的标准化：不改变模型能力，只是把工具接入从一次性胶水代码变成可复用的生态组件。对个人开发者，它省掉大量重复集成；对团队，它让工具层和 Agent 层可以分开演进。如果你在做 OpenClaw 插件或自动化流程，值得花一个下午把 stdio 模式跑通——之后的扩展就全是增量了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/f6c89c355b4a946a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/eff89e678faa9685.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/359038b28296b9dd.png)

