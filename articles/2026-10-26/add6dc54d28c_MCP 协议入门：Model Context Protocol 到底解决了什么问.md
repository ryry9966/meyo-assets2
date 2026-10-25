---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42588
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

做 Agent 自动化，最耗时的往往不是模型侧，而是"接线"：让 Agent 读到你的数据库、调用内部 API、操作文件系统。每个框架一套插件体系，每个数据源一个定制 adapter，写多了发现全是重复劳动。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），把"模型应用如何连接外部工具与数据"这件事标准化。目前主流宿主（Claude Desktop、Cursor、各类 Agent 框架）基本都已支持，值得花一小时搞清楚它是什么、不是什么。

## 它解决了什么问题

一句话：把 M×N 的集成问题降成 M+N。

- M 个 Agent 宿主 × N 个工具/数据源，点对点对接要写 M×N 份胶水代码；
- 有了 MCP，工具方实现一次 Server 即可被所有兼容宿主发现和调用，宿主方实现一次 Client 即可接入所有 Server。

MCP 基于 JSON-RPC 2.0，核心抽象三类能力：

- **Tools**：模型可主动调用的函数（有副作用）；
- **Resources**：可读取的上下文数据（只读）；
- **Prompts**：预置的提示词模板。

传输层常用两种：本地进程走 stdio，远程服务走 Streamable HTTP。协议本身不负责鉴权和业务逻辑，这些留在 Server 侧。

换句话说，MCP 是"USB-C 接口"，不是"万能驱动"。它不提升模型能力，只让能力可插拔。

## 跑通一个最小链路

1. 选一个支持 MCP 的宿主（OpenClaw、Claude Desktop、Cursor 均可）；
2. 配置文件里注册一个现成 Server，例如官方 filesystem：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/work"]
    }
  }
}
```

3. 重启宿主，确认工具列表出现 `read_file` / `write_file` 等；
4. 用 MCP Inspector（`npx @modelcontextprotocol/inspector`）单独调试 Server，不依赖宿主；
5. 需要内部能力时，用官方 Python/TypeScript SDK 写 Server：装饰器暴露函数、写清 docstring，十几行就能跑。

## 踩坑点

- **工具描述就是给模型看的 API 文档**。描述含糊，模型就会选错工具或乱传参。我吃过亏：一个 search 工具没写清返回条数上限，模型默认拉全量，把上下文撑爆。
- **stdio 型 Server 对启动环境敏感**：command 建议用绝对路径，Windows 下 `npx` 可能需要 `cmd /c` 包一层。
- **规格还在演进**：2025-03 版已用 Streamable HTTP 取代 SSE，鉴权方案也在调整，旧教程部分写法已失效，以官方 spec 为准。
- **工具不是越多越好**：挂几十个 Server、上百个工具，模型选择准确率明显下降，还浪费 token。按任务场景挂载。
- **安全别糊弄**：Server 以你的身份运行，第三方 Server 等于供应链风险；Resource 内容会进入上下文，存在提示注入面。最小权限、目录白名单、来源可信，一个都不能省。

## 可复用的建议

- 先用现成 Server（filesystem、git、fetch）验证链路，再考虑自研；
- 自研前先问：这个能力会被多个宿主/场景复用吗？只有一次性需求，写个脚本让 Agent 用 shell 调用反而更简单；
- 一个 Server 只管一个领域，保持小而清晰；
- 给每次工具调用留日志——排查"模型为什么这么调"时，这是唯一可靠的证据；
- 锁定 Server 版本，升级前在 Inspector 里回归一遍工具列表和参数 schema。

## 总结

MCP 解决的是集成成本问题，不是智能问题。它把"每接一个工具写一份胶水"变成"各自实现一次协议"：工具作者写一次即可被所有兼容宿主使用，Agent 开发者把接能力从开发问题降级成配置问题。理解了这一点，就知道什么时候该上 MCP，什么时候一个脚本就够了——这是社区里最常被问、也最值得先想清楚的问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/a71a89caec42eb4c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/d1e57bba32558cee.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/7c4b945df9c71480.png)

