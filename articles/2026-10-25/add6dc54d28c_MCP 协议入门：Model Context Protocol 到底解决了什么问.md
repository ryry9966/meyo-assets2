---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42541
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

Function calling 普及之后，给模型挂工具这件事变成了体力活：OpenClaw 要接文件系统，接浏览器，接内部 API，每接一个都要写一层胶水代码——发现工具、传参、鉴权、错误处理，每个宿主框架的插件格式还不一样。工具方要为每个 agent 平台维护一份适配，宿主要逐个接入。2024 年底 Anthropic 提出 MCP（Model Context Protocol），一年多下来已经成了这个领域事实上的通用协议，OpenClaw 也原生支持。

## 它到底解决了什么问题

一句话：把 M×N 变成 M+N。M 个 agent 框架、N 个工具，以前是两两写集成；有了协议，工具方实现一次 MCP server，任何支持协议的 client 都能直接用。

更深一层看，function calling 只回答"模型能不能调工具"，MCP 管的是调之前的一整套工程：工具发现（tools/list）、传输方式（本地 stdio / 远程 Streamable HTTP）、进程生命周期、鉴权（OAuth）、上下文资源（resources / prompts）。它把"接入工具"从一个代码问题降级成一个配置问题。

## 在 OpenClaw 里跑通最小示例

1. **选 transport**：本地工具用 stdio（框架拉起子进程），远程服务用 HTTP。
2. **注册 server**（示意，字段以当前版本文档为准）：

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

3. **重启并确认工具被发现**：看日志里的工具列表，而不是想当然。
4. **权限收口**：只挂需要的目录，新 server 先只读。
5. **用一个具体任务验证**：比如"列出该目录下最近修改的三个文件"，能正确选工具、传对参数，才算通。

## 踩坑点

- **stdout 污染**：stdio 模式下 stdout 就是 JSON-RPC 通道，server 里随手 `print` 一行日志，协议直接断。日志一律写 stderr。
- **description 是 prompt，不是注释**：写得含糊，模型就乱调、漏参。把单位、约束、副作用写进去，标准是"新人照着能调对"。
- **工具数量失控**：几十个 tool 全量塞进 context，token 先爆，模型选择准确率也降。按需启用、拆 server、粗粒度封装。
- **进程生命周期**：stdio server 是子进程，崩溃重启、僵尸进程、状态丢失都要有预案，别当成常驻服务。
- **MCP ≠ 沙箱**：filesystem server 授权了路径就等于给了读写权，真正的鉴权和审计在 server 侧自己实现。

## 可复用建议

- 一个内部系统封装成**一个** server，暴露 3~5 个粗粒度工具，别一接口一工具。
- description 当接口文档写，附一个典型调用示例，命中率明显提升。
- 团队维护一份内部 server 清单：transport、权限范围、owner、版本，避免重复造轮子。
- 写操作灰度：只读跑一周，确认行为稳定再放开写。

## 总结

MCP 不是让模型变聪明的魔法，它只是把"agent 接工具"标准化了。价值在解耦和生态：工具方写一次，宿主任意换，社区现成 server 直接复用。对 OpenClaw-CN 里自己攒 agent、自己接自动化的玩家来说，省下的胶水代码和维护成本，就是它全部的意义。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/0e7e470df0160b82.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/524591e47194f840.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/f6e308ee665aea5e.png)

