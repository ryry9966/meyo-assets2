---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38352
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

过去两年做 Agent 开发的人，基本都在重复同一件事：让模型调用外部工具。文件读写、数据库查询、内部 API……每换一个框架、每接一个数据源，都要重写一遍胶水代码。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标是把「模型应用」和「工具/数据提供方」之间的对接标准化。它不决定模型怎么思考，只规定双方用什么语言说话。

## 它到底解决什么问题

核心是 **M×N 集成问题**：M 个应用 × N 个数据源，原本需要 M×N 个定制连接器。有了 MCP，数据源方实现一次 Server，应用方实现一次 Client，即可互通——类似 USB-C，接口统一了，设备随便换。

协议本身不重：基于 JSON-RPC 2.0，支持 stdio（本地子进程）和 Streamable HTTP（远程）两种传输。Server 对外暴露三类原语：

- **Tools**：模型可主动调用的动作（查库、发请求）
- **Resources**：可读取的上下文数据（文件、记录）
- **Prompts**：预置的提示词模板

## 做法：三步跑通

1. **选一个支持 MCP 的宿主（Host）**。OpenClaw、主流 IDE 插件和桌面助手都已支持，配置通常是一个 JSON 块。
2. **挂一个现成 Server 验证链路**。以 filesystem 为例：

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

重启宿主后，Client 会自动 `tools/list` 拉取工具清单，模型在对话中即可调用。

3. **需要定制时，用官方 SDK（Python/TypeScript）写自己的 Server**：定义函数、写好入参 schema 和 description、注册为 tool，十几行就能跑起来。

## 踩坑点

- **description 是写给模型看的，不是写给人看的**。「查询用户」不如「按手机号或邮箱查询用户资料，返回最近一条订单」。描述含糊，模型就瞎选工具，这是最常见的翻车原因。
- **工具数量失控**。挂七八个 Server、上百个工具后，上下文膨胀，选择准确率明显下降。按需启用，用不到的先关。
- **安全边界要想清楚**。本地 stdio Server 以你的用户权限运行，能读你读的一切。第三方 Server 装之前看代码，敏感场景加沙箱；写操作默认要求确认。
- **传输方式别选错**。本地工具用 stdio 简单可靠；远程 Server 要处理鉴权和超时，长任务记得支持进度上报。
- **规范还在演进**（2024-11 到 2025-06 多个版本），Server 与 Client 版本不匹配会出现工具列表为空等玄学问题，先对齐版本再排查。

## 可复用建议

- 把 MCP Server 当微服务设计：工具面收窄、命名加前缀防冲突、操作尽量幂等。
- 先做只读工具，跑稳了再放写操作，写操作带二次确认。
- 密钥走环境变量，别提交进配置仓库。

## 总结

MCP 没有引入黑魔法，它做的事是「约定接口」。对个人开发者，它省掉了大量胶水代码；对团队，它让工具资产能在不同 Agent 之间复用。建议从挂一个官方 Server 开始跑通全链路，再评估是否值得为内部系统写 Server——多数情况下，这一步是值得的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/b99757362497144f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/2c84f420a0d52cbc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/cb167234fb4e8ece.png)

