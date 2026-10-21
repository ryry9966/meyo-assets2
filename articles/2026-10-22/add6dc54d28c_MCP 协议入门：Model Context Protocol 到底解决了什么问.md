---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42235
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

过去一两年，把工具接进 LLM 这件事基本是各自为战：Function Calling 的 schema 各家略有差异，每个 Agent 框架都发明了一套自己的插件格式。你写了一个"查数据库"的工具，在框架 A 里要包一层胶水，换到框架 B 又得重写。工具本身没变，变的是接入层。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），试图把这层胶水标准化。它本质上是一个开放协议，定义了 LLM 应用与外部工具、数据源之间的通信规范。

## 它到底解决了什么问题

核心是 M×N 问题：M 个 Agent 应用 × N 个工具，过去需要 M×N 次适配。有了 MCP，工具方只需实现一次 MCP Server，任何支持 MCP 的客户端都能直接使用，问题变成 M+N。

具体拆开，MCP 统一了三件事：

1. **工具发现**：客户端通过 `tools/list` 拿到工具清单和参数 schema，不用硬编码；
2. **调用约定**：统一的 `tools/call` 请求/响应格式，错误语义清晰（业务失败用 `isError` 标记，区别于传输层异常）；
3. **传输方式**：stdio（本地子进程）和 Streamable HTTP（远程服务）两种标准传输，HTTP 模式下走 OAuth 鉴权。

对实践者来说最大的收益是复用：社区现成的 MCP Server（文件系统、Git、数据库、浏览器）可以直接挂载，省掉自己写胶水的时间。

## 在 OpenClaw 里怎么接

1. 确认所用版本的 MCP 支持已开启（配置中对应 `mcpServers` 段）；
2. 本地工具走 stdio：指定 server 的启动 command 和 args，例如官方 filesystem server；
3. 远程工具走 URL：接入 Streamable HTTP server；
4. 重启后确认工具列表已加载，在对话里触发一次调用做验证；
5. 控制挂载数量，只挂当前场景真正需要的 server。

## 踩坑点

- **工具不是越多越好**。每个工具的 schema 都占上下文，挂几十个 server、上百个工具后，模型选错工具的概率明显上升。按项目裁剪。
- **description 要当 prompt 写**。自研 MCP Server 时，工具描述含糊，模型就会乱传参数。这是新手最常见的翻车点。
- **stdio 的进程管理**。客户端退出时子进程不一定干净回收，长会话注意僵尸进程和句柄泄漏。
- **安全边界**。挂第三方 MCP Server 等于授予执行权限，尤其 filesystem、shell 类。先确认来源和最小权限，别全量挂载。
- **isError 与异常要分清**。工具执行失败应返回 `isError: true` 让模型自行重试或换路，直接抛异常会中断整轮调用。

## 可复用建议

- 把常用 server 组合按场景做成配置片段，比如"写代码套件"（filesystem + git + 搜索），新项目直接复用；
- 团队自研内部系统时，优先暴露 MCP 而非只提供 REST API，一次投入，所有 Agent 受益；
- 给每个 MCP Server 维护一份最小权限清单：默认只读，写操作显式开启。

## 总结

MCP 没有让工具变强，它做的是把"接入"这件事的成本打下来。对个人用户，意味着现成生态即插即用；对团队，意味着工具层和 Agent 层可以解耦演进。如果你的自动化流程里还留着手写的胶水代码，值得花一个下午迁到 MCP 上——不是因为它新，而是因为之后不用再写第二遍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/aa84235f1f4d2c99.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/0e1d1eddb21f7bec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/bfe8fcf93f024d9f.png)

