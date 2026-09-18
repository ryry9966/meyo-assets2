---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38072
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

过去一年做 Agent 的人基本都在重复同一件事：让模型连上外部系统——查数据库、读工单、操作浏览器、调内部 API，每接一个工具就得为当前框架写一层胶水代码。2024 年底 Anthropic 开源了 Model Context Protocol（MCP），把这条"最后一公里"标准化。目前主流客户端和多数 Agent 框架都已支持，它正在成为工具接入的事实接口。

## 它解决了什么问题

一句话：把 M×N 的集成问题变成 M+N。

没有 MCP 时，M 个宿主应用要对接 N 个工具/数据源，理论上需要 M×N 套适配代码，而且各家插件接口互不兼容，写完不可迁移。MCP 用基于 JSON-RPC 2.0 的协议把两端解耦：工具方实现一次 Server 即可被所有宿主使用；宿主实现一次 Client 即可接入所有现成 Server。顺带统一了三件事：工具发现、调用约定（参数 schema）、上下文注入。

需要泼一盆冷水：MCP 不提升模型能力，也不解决提示词工程问题，它只是一份接口契约。价值在复用，不在"变聪明"。

## 协议结构与上手步骤

三个角色：

- **Host**：Agent 宿主，桌面客户端或你自己的 Agent 进程；
- **Client**：Host 内部的连接器，与 Server 一对一通信；
- **Server**：能力提供方，暴露三类原语——tools（模型调用）、resources（注入上下文）、prompts（提示模板）。

两种传输：**stdio**（本地子进程，适合开发调试）和 **Streamable HTTP**（远程部署、团队共享；旧版 SSE 双通道传输已被规范废弃，选型时注意版本）。

上手建议三步：

1. 先跑现成 Server。filesystem、git、postgres、playwright 等官方维护的 Server 覆盖大部分场景，配置通常就是一段 JSON：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/data"]
    }
  }
}
```

2. 在你的 Agent 里 list 一次 tools，看模型实际拿到的描述长什么样——这直接决定调用质量。
3. 确认有必要后再自研 Server。包一层内部 API、读内部库这类需求，用官方 SDK 一两百行就能跑起来。

## 踩坑点

- **stdio Server 的日志别打到 stdout**：stdout 是 JSON-RPC 通道，一条杂音日志就能让整个连接解析失败，日志一律走 stderr。
- **工具描述就是新的提示词**：描述含糊，模型要么不调、要么乱传参。写完工具务必用真实任务回归测试，别只看 schema 对不对。
- **参数 schema 越简越好**：深嵌套、多必填会显著拉高调用失败率，也吃上下文窗口；一个宿主挂几十个工具时，光描述就可能占掉不少 token。
- **安全上别裸奔**：MCP Server 本质是给模型一段可执行代码，第三方 Server 等于供应链依赖，装之前过一遍源码；涉及写操作默认不要 auto-approve，加人工确认。
- **规范仍在演进**：传输层、鉴权都在变，客户端和服务端版本不匹配是远程部署最常见的故障源，部署时锁定版本。

## 可复用建议

- 能用现成 Server 就不自研；自研优先包内部 API，别重复造 filesystem 这类轮子。
- 工具设计成粗粒度、单一职责，返回结构化、对模型友好的结果，别吐一坨原始 HTML。
- 本地开发用 stdio，团队共享用 Streamable HTTP + 网关统一做鉴权与审计。
- 给"模型是否正确调用"建一个小评测集，工具改动就跑一遍，比肉眼盯对话可靠得多。

## 总结

MCP 解决的不是模型问题，而是集成问题：用一份开放契约替换掉各自为政的插件接口，让工具生态第一次可以在不同宿主之间复用。它朴素、甚至有点无聊——JSON-RPC、schema、进程通信，都是老东西。但对做 Agent 和自动化的人来说，"无聊且统一"恰恰是基础设施该有的样子。建议从挂一个现成 Server 开始，跑通一条真实业务链路，再决定投入多深。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/4acf0ae6d29e5199.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/9c91dbd58179e9d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/4ec78718ab8b9a65.png)

