---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41216
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

Agent 能力上不上得去，卡点往往不在模型，而在"接工具"这一层。你想让自动化 Agent 读本地文件、查数据库、调内部 API，每种接法都要写一遍胶水代码；换个宿主框架、换个模型，还得再写一遍。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底发布的开放协议，一句话概括：给模型和外部工具/数据之间定一个标准插座。目前 Claude Desktop、主流 IDE 和多数 Agent 框架已支持客户端角色，生态里也积累了不少可直接复用的 MCP Server。

## 它到底解决什么问题

经典问题是 M×N：M 个 Agent 宿主 × N 个工具，理论上要写 M×N 份集成代码。每个宿主有自己的 tool calling 格式，每个工具有自己的接入方式，全靠胶水代码堆。

MCP 把它压成 M+N：宿主实现一次 MCP Client，工具实现一次 MCP Server，中间用 JSON-RPC 2.0 通信。协议定义了三类原语：

- **Tools**：模型可调用的动作（查询、写入、执行）
- **Resources**：可注入上下文的数据（文件、日志、表结构）
- **Prompts**：预置的提示词模板

## 跑通最小路径

以最简单的 stdio 方式为例，三步：

1. 选一个现成 Server（比如文件系统）挂到宿主配置里：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    }
  }
}
```

2. 重启宿主，确认工具列表出现在会话里。
3. 用自然语言触发一次调用，检查返回结构能否被模型正确理解。

跑通之后再考虑自己写：官方 SDK（Python/TypeScript）十几行就能包一个 Server。把已有的 CLI 脚本直接暴露成 Tool，通常比重写划算。

## 踩坑点

- **stdio Server 继承宿主环境变量**。Node/Python 版本不一致是最常见的"起不来"原因，先在终端手动跑一遍命令再排查配置。
- **Tool 描述就是给模型看的文档**。描述含糊，模型就选错工具、传错参数——很多"MCP 不工作"其实是描述写得不行。
- **别把 30 个工具全挂上试试就知道了**：上下文膨胀，命中率骤降。按场景拆配置，用哪个挂哪个。
- **传输方式有讲究**：stdio 简单但只能本地；Streamable HTTP 适合远程，但涉及 OAuth 2.1 鉴权，部分宿主支持还不齐，混合环境要提前验证。
- **安全别裸奔**：MCP Server 以你的本地权限运行，来源不明的第三方 Server 等于交出 shell；Resource 返回内容会被注入上下文，警惕间接提示注入。

## 可复用建议

- 先 stdio、先少量工具、先复用现成 Server，跑通再加复杂度。
- Tool 描述按"给模型看的 API 文档"标准写：做什么、参数含义、什么场景该选它。
- 读写分离，危险操作单独成 Tool，并在描述里声明需要确认。
- 锁定 SDK 版本。协议还在迭代，注意 protocol version 的兼容声明。
- 把内部脚本包成 Server，而不是迁到某个平台，保留随时撤出的能力。

## 总结

MCP 解决的不是模型能力问题，而是集成层问题：把点对点的胶水代码收敛成标准接口。它不会让你的 Agent 突然变聪明，但能让你在换宿主、换模型、加工具时少写九成胶水代码。对做自动化和插件生态的人来说，它是目前值得押注的一层标准。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/8fffadc6578c9e63.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/5987d686bd9b04cb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/14660055a3af038e.png)

