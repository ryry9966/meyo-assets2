---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 39084
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景：先说 M×N 这个老问题

在 MCP 出现之前，给 Agent 接工具基本是手工艺活。OpenClaw 这类网关要接文件系统、浏览器、数据库、内部 API，每个目标都得写一份定制插件；换一个宿主（IDE、桌面助手、CI 机器人），同样的集成又要重来一遍。M 个应用 × N 个工具 = M×N 份胶水代码，而且没有一份能复用。

MCP（Model Context Protocol）是 Anthropic 在 2024 年 11 月开源的协议，基于 JSON-RPC 2.0，目标很朴素：把「模型应用如何获取上下文和工具」这件事标准化。

## 它到底解决什么

MCP 不会让模型变聪明，它解决的是契约问题。协议核心就三个原语：

- **tools**：模型可调用的操作（函数）
- **resources**：应用侧可控的数据（文件、记录）
- **prompts**：用户侧可复用的模板

架构分三层：Host（如 OpenClaw、IDE）内嵌 Client，每个 Client 与一个 Server 保持 1:1 连接；握手时双方协商 capability，之后按需调用。传输层两种：本地进程走 stdio，远程服务走 Streamable HTTP。

一句话总结：M×N 的集成问题变成 M+N——Server 写一次，所有支持 MCP 的 Host 都能用；Host 实现一次，所有 MCP Server 都能挂。

## 怎么起步（5 步）

1. **选传输方式**：本地工具选 stdio，远程服务选 Streamable HTTP。
2. **用官方 SDK**（TypeScript / Python 均可）起一个最小 Server，注册一两个工具，参数用 JSON Schema 描述。
3. **在 Host 侧配置** Server（命令或 URL），重启后确认握手成功、工具列表可见。
4. **认真写 description**——这是给模型看的选型依据，不是给人看的注释。
5. **用 MCP Inspector** 或直接抓 JSON-RPC 日志验证调用链路。

OpenClaw 侧的配置大致长这样：

```json
{
  "mcpServers": {
    "weather": {
      "command": "npx",
      "args": ["weather-mcp-server"]
    }
  }
}
```

## 踩坑点

1. **description 写成 API 文档**。模型靠它选工具，`query(city: str)` 这种描述没用，要写清何时该用、失败时返回什么。这是工具被选错的头号原因。
2. **工具数失控**。一个 Server 塞 40 个工具，工具列表直接撑爆上下文，选择准确率也会掉。按领域拆 Server，默认只启用需要的。
3. **stdio 的进程问题**。Server 是 Host 的子进程，崩溃即断连；Windows 下还有编码坑，中文输出乱码多半是子进程没用 UTF-8。
4. **版本漂移**。协议在 2024-11、2025-03、2025-06 三版迭代很快，老 Client 只支持 SSE、新 Server 用 Streamable HTTP 会对不上，SDK 版本务必对齐。
5. **安全别省**。Server 拿着你本地凭据运行，第三方 Server 是供应链风险；工具返回内容还可能夹带注入指令。先接只读工具，写操作务必加确认层。

## 可复用建议

- 把 description 当 prompt 写，而不是当注释写。
- 错误返回模型可读的文本，别直接抛异常——模型需要从错误里自己恢复。
- 全量记录 JSON-RPC 往返日志，排障效率差一个数量级。
- 单 Server 工具数控制在个位数到十几个，宁多拆勿堆砌。

## 总结

MCP 的价值不在「又多了一种协议」，而在于把 Agent 生态里最脏、最重复的集成层抽成了标准接口。对 OpenClaw 这种聚合多插件、多数据源的宿主来说，收益最直接：一个协议，接一次，到处可用。建议从只读工具起步，把 description 打磨好，再逐步放开写操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/1d9a0dc4bb2dd757.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/cf65cbf1b1073c8b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/bef0e8f7bd33136f.png)

