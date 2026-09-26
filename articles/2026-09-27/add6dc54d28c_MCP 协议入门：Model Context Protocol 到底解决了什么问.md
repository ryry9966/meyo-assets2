---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 39170
source: 综合讨论
publishedAt: 2026-09-27
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年多下来，它基本成了 Agent 生态里连接外部工具和数据的事实标准。但很多人第一次接触都会问：Function Calling 不是早就有了吗，为什么还需要一个“协议”？

## 问题：M×N 的集成成本

看没有 MCP 时的状态就明白了。

每个 Agent 框架有自己的工具/插件定义格式，每个外部系统（数据库、SaaS、内部 API）都要为每种框架单独写适配。M 个客户端 × N 个数据源，就是 M×N 份胶水代码。换一个 Agent 框架，之前接的工具全部重写。

MCP 把 M×N 压成 M+N：客户端实现一次协议，就能对接所有标准化 server；每个数据源只暴露一个 server，就能被所有客户端使用。类比 USB-C——外设不用为每台电脑做专用接口。

另一个容易忽略的点：MCP 不只是“调工具”。它定义了三类原语——Tools（模型可调用）、Resources（可读取的上下文数据）、Prompts（预置模板），把“给模型塞上下文”这件事也标准化了。

## 结构速览

- **Host**：Agent 运行时，比如 OpenClaw 的 agent 进程
- **Client**：Host 内部与单个 server 的 1:1 连接
- **Server**：暴露能力的进程；本地常见形态是 stdio 子进程，远程走 Streamable HTTP

## 上手步骤

1. 确认你的 host 支持 MCP（OpenClaw 侧已支持声明式配置）。
2. 从现成 server 开始，比如 filesystem 或某个内部服务的封装，别上来就自己写。
3. stdio 形态的配置各家大同小异：

```json
{
  "mcpServers": {
    "notes": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./notes"]
    }
  }
}
```

4. 用 MCP Inspector 或 host 的调试模式，先看 server 暴露了哪些 tool，确认 schema 可读。
5. 跑一个最小任务，观察模型是否选对工具、参数是否正确，再逐步加东西。

## 踩坑点

- **环境问题占一半排障时间**。stdio server 是子进程，Python venv、Node 版本、PATH、环境变量都和你 shell 里的不一致。Windows 上跑 `npx` 常要包一层 `cmd /c`。
- **工具描述就是给模型看的 prompt**。name/description 写得含糊，模型就会选错工具或传错参数。这不是模型的问题，是你没写好文档。
- **token 成本**。挂几十个 server，光工具定义就能吃掉大块上下文。工具数量要克制，按 agent 的实际任务裁剪。
- **权限别放飞**。server 以你的身份运行，工具返回结果里可能被注入指令（tool poisoning）。写操作默认要人工确认，别无脑 auto-approve。
- **规范还在演进**。传输层已从 SSE 迁移到 Streamable HTTP，老 server 记得升级。

## 可复用建议

- 先复用社区 server，把模式跑通，再封装自己的内部 API。
- 自研 server 遵循最小权限：先只读，验证后再放开写。
- 每次工具调用留日志，出问题能回放定位。
- 工具粒度对齐任务粒度：一个“查询订单”比一个“执行 SQL”对 agent 友好得多。

## 总结

MCP 没有发明新能力，它做的是把 Agent 接入外部世界的接口标准化，把集成成本从 M×N 降到 M+N。要不要用，判断标准很简单：如果你的工具或数据要被不止一个 agent、客户端消费，MCP 就值得接；如果只是单脚本一次性调用，直接写 Function Calling 反而更快。工程上把它当 USB-C 用，别当银弹。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-27/4567078444f79623.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-27/f7261843e83666cf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-27/6d075bf50c76af07.png)

