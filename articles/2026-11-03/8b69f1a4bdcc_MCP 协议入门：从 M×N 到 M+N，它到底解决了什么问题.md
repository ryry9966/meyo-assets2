---
title: MCP 协议入门：从 M×N 到 M+N，它到底解决了什么问题
feedId: 43308
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年多时间它基本成了 Agent 生态里工具接入的事实标准。OpenClaw 这类本地 Agent 框架、各家 IDE 和桌面客户端都陆续把"接 MCP server"做成一等公民能力。但在动手配 `mcpServers` 之前，值得先想清楚：MCP 到底解决了什么、没解决什么。

## 它解决的是 M×N 问题

MCP 之前，模型调用外部能力靠的是各家自有的 function calling / plugin 机制。假设有 M 个 AI 应用、N 个工具（数据库、浏览器、内部 API、日历……），最坏情况要维护 M×N 份胶水代码：每个应用为每个工具单独实现鉴权、schema、错误处理。工具方要适配 N 套 SDK，应用方要追着 N 个接口改。

MCP 把乘法变成加法：应用实现一次 MCP client，工具实现一次 MCP server，双方按同一份协议协商能力。官方类比是 USB-C——接口统一了，设备随便插。

协议本身不复杂，核心是三类原语：

- **Tools**：模型可主动调用的动作（查库、发消息、跑脚本）；
- **Resources**：应用可读取的上下文数据（文件、表结构、文档）；
- **Prompts**：预置的提示模板。

传输上，本地进程走 stdio，远程走 Streamable HTTP。身份、重试、复杂编排协议都不管——这是它克制的地方。

## 实际接一个：四步

以给本地 Agent 接 filesystem 加一个自建 API 的 server 为例：

1. **选 server**。优先官方或社区活跃的实现（filesystem、git、postgres 都有现成的）。自建场景用官方 SDK 把内部 REST API 包一层，几十行就够。
2. **写配置**。stdio 方式为例（OpenClaw 及多数客户端结构类似）：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/me/projects"]
    }
  }
}
```

3. **验证工具列表**。重启后确认工具已注册，用 MCP Inspector 或手动发一次调用，确认返回结构符合预期。
4. **跑一条真实任务**。比如"列出项目里所有 TODO 整理成周报"，观察模型选了哪个工具、参数对不对。端到端通了才算接完。

## 踩坑点

- **工具越多，模型越笨**。每个工具 schema 都会进上下文，挂十几个 server 后 token 成本上升，选错工具的概率也明显上升。只开当前任务需要的 server。
- **工具描述就是给模型看的 API 文档**。自建 server 时 `description` 写得含糊，模型就会不调或瞎调。按"什么场景该用我、参数什么含义"来写，别照抄函数注释。
- **stdio 的环境继承**。server 进程继承 Agent 的环境变量，`PATH` 不全会导致 spawn 静默失败；密钥写进配置文件要小心仓库泄露，尽量走 env 引用或 secret 管理。
- **版本与传输迁移**。早期基于 HTTP+SSE，2025 年规范切到 Streamable HTTP。新旧混接时版本协商失败的典型表现是"工具列表为空"，而不是报错，很容易误判为没装上。
- **安全别省**。MCP server 拿着你的本地凭证，工具返回内容又会被模型读取——一段注入了指令的网页内容理论上能诱导模型调用写操作。先只读、后放开写，写操作加人工确认。

## 可复用建议

- 评估第三方 server 看三点：工具粒度是否合理（一个工具干一件事）、返回是否支持过滤和分页、维护是否活跃。
- 自建时面向模型设计接口，而不是把 REST API 原样翻译；宁可多个窄工具，不要一个万能工具。
- 把 server 清单放进版本管理，团队直接复用配置，换框架时工具层零成本迁移。

## 总结

MCP 没有让模型更聪明，它解决的是"连接"这一层的标准化：工具写一次，应用接一次。它也不管 Agent 会不会规划任务、上下文管理好不好——那些仍是应用层的功课。但正如 USB-C，标准的价值随接入数量复利。对做自动化的实践者来说，现在值得做的就是把手里的高频内部能力包成 MCP server，让以后每次换模型、换框架时，工具层不必重写。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/73a6c5938589237f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/b9c1bc58842ba2a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/7bd8bc4e18177536.png)

