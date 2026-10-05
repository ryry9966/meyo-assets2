---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 40587
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景：M×N 的接入困境

做过 Agent 自动化的人多半经历过这样的场景：模型侧，各家 function calling 的 schema 格式不完全兼容；工具侧，每个内部系统都要单独写描述、鉴权、参数解析。3 个 Agent 框架接 10 个数据源，理论上就是 30 份胶水代码，而且模型或框架一换版本，工具层就要跟着重写。

MCP（Model Context Protocol）是 Anthropic 2024 年底开源的协议，目标只有一个：把 M×N 压成 M+N。数据源方实现一次 MCP Server，任何支持 MCP 的 Host——OpenClaw、IDE、各种 Agent 运行时——都能直接复用。

## 它到底解决了什么

先说它不解决什么：MCP 不会让模型更聪明，也不负责编排和记忆。它解决的是"集成"这一层的标准化。

协议构建在 JSON-RPC 2.0 之上，定义了三类原语：

- **tools**：模型可主动调用的动作，比如查库、发消息、跑脚本；
- **resources**：Host 可读取的上下文数据，比如文件、表结构；
- **prompts**：可复用的提示模板。

传输层分两种：本地进程用 stdio，远程服务用 Streamable HTTP（旧实现是 HTTP+SSE）。工具如何被描述、发现、调用、鉴权，协议都给了统一答案——这正是以前每个插件各自造轮子的部分。实际使用中，OpenClaw 这类自动化场景主要消费的是 tools，resources 在部分 Host 里支持还不完整，别默认两边对等。

## 在 OpenClaw 里跑通一个 Server

1. **认清三个角色**。Host 是运行时（OpenClaw）；Client 是 Host 内与单个 Server 一对一会话的连接器；Server 是暴露 tools/resources 的进程。
2. **先用现成的**。filesystem、fetch、playwright、sqlite 等官方与社区实现，足够覆盖大部分自动化需求，不建议上来就自研。
3. **写配置**。stdio 类 Server 通常只需声明命令和参数：

```json
{
  "mcpServers": {
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"]
    }
  }
}
```

具体字段名以你当前版本的 OpenClaw 文档为准，这里只示意结构。

4. **先调试再接入**。用 MCP Inspector（`npx @modelcontextprotocol/inspector`）单独跑 Server，确认工具列表、参数 schema、手动调用都正常，再让 Agent 上。
5. **观察实际行为**。工具注入后，看模型在真实对话里会不会调、调得对不对，再决定收窄还是扩充工具集。

## 踩坑记录

- **stdout 污染**。stdio Server 往 stdout 打任何日志都会直接破坏 JSON-RPC 流，日志必须走 stderr。这是翻车率最高的一条。
- **描述即 prompt**。工具 description 含糊、参数语义不清，模型要么不敢调，要么参数乱填。写描述要认真，本质是 prompt 工程。
- **工具过多**。几十个工具全开，上下文膨胀且选择准确率下降，按任务启用子集。
- **凭据管理**。远程 Server 的 token 走环境变量注入，别明文进仓库。
- **Windows 的 stdio**。npx 在 Windows 上实际是 .cmd shim，直接 spawn 可能失败，需要 shell 包装或改用 node 直调。

## 可复用的建议

- 先复用再自研；自研时一个 Server 只做一件事，工具粒度偏"任务级"而不是"函数级"。
- 把 MCP Server 当独立小服务对待：schema 严格校验、版本化、副作用在 description 里写明。
- MCP 的核心红利是"换 Host 不换 Server"，设计时保持 Server 与任何 Host 解耦，别把 OpenClaw 特有逻辑塞进去。

## 总结

MCP 把"接入一个新工具"从项目级定制变成了配置级声明。它不是模型能力的革命，而是一件务实的基础设施：先用好现成 Server，把工具描述写扎实，把工具数量控制住，就能拿到八成收益。剩下的部分，等真有定制需求再去自研也不迟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/58f2f9adbee05de5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/e887184ed2a210c2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/24ec7bed7f773cd6.png)

