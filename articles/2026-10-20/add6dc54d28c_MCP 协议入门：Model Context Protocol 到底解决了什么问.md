---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42053
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

过去两年做 Agent 自动化，最耗时间的往往不是提示词，而是"接线"：让模型能读文件、查数据库、调脚本。各家模型厂商的 function calling 格式互不兼容，工具写一遍只能在自家生态里用。2024 年底 Anthropic 把 MCP（Model Context Protocol）开源出来，思路很朴素：把"模型怎么连外部能力"标准化成协议，谁实现谁受益。

## 它解决的问题：从 M×N 到 M+N

没有 MCP 时，M 个 Agent 应用要接 N 种工具，理论上要写 M×N 份适配代码。MCP 把两边拆开：应用侧实现 MCP 客户端，工具侧实现 MCP 服务端，中间用统一的 JSON-RPC 消息通信，集成成本变成 M+N。

日常打交道最多的是三个原语：

- **Tools**：模型可主动调用的动作（写文件、发消息）；
- **Resources**：应用侧可读取的数据（文件、配置、查询结果）；
- **Prompts**：预置的提示词模板。

架构上是经典 client-server：宿主（比如 OpenClaw 这类 Agent 运行时）内嵌 client，每个 MCP server 是独立进程，通过 stdio（本地子进程）或 Streamable HTTP（远程）传输。注意：老教程里的 SSE 传输已标记废弃，新实现直接用 Streamable HTTP。

## 实操：接入一个 MCP server

以接文件系统 server 为例：

1. 确认宿主有 MCP 配置入口（OpenClaw 在 skills/工具配置里声明 MCP server）；
2. stdio 模式配置 `command` + `args`，指向 server 启动命令；
3. 重启后让宿主枚举工具，确认工具列表已注入模型上下文；
4. 用一个低风险任务实测调用，检查参数是否按 schema 传入；
5. 查调用日志，确认返回内容被正确回填。

全程不到十分钟，但第 4、5 步不要跳过。

## 踩坑点

- **工具描述质量决定调用质量**。description 含糊，模型就瞎猜参数或该调不调。这部分是人写、模型读的接口文档，值得花时间打磨。
- **工具越多，上下文越贵**。十几个 server 全开，工具定义能吃掉几千 token，还会互相干扰。按任务启用，别贪多。
- **stdio server 挂住会拖垮宿主**。子进程阻塞、调试日志混进 stdout 导致 JSON-RPC 解析失败，都是高频故障。加超时，调试输出走 stderr。
- **安全上不要裸奔**。装第三方 MCP server 等于交出部分执行权，返回内容里也可能夹带注入指令。权限白名单、只读优先、写操作加人工确认，三件套缺一不可。

## 可复用建议

- 自研 server 时一个工具只做一件事，命名用"动词+对象"，参数带校验；
- 优先复用社区验证过的 server，只对缺口自研；
- 每次工具调用落日志（工具名、参数、耗时、结果摘要），排查全靠它；
- 先在本地 stdio 跑通，再考虑远程部署，不要一步到位。

## 总结

MCP 没有让工具变强，它只是把接线的口子统一了。真正决定 Agent 好不好用的，仍是工具设计的克制程度和安全边界。把协议当胶水用，把精力花在工具质量和权限控制上——这是用下来最实在的结论。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/381483c8237235f7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/ec84d83471467375.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/bf82250442e86627.png)

