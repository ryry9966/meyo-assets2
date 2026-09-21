---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38329
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景：老问题的标准化时机

做 Agent 自动化的人都会撞上同一堵墙：模型的推理能力不缺，缺的是"够得着"的数据和动作。让 Agent 查数据库、发消息、操作文件，过去只能为每个框架手写插件——这个框架一套接口，那个 Agent 另一套，同一个 GitHub 集成可能要写三遍。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），定位很朴素：给模型和外部工具之间定一个标准接口，官方类比是 USB-C。目前主流客户端和官方 SDK（Python / TypeScript）都已跟进，值得花一小时搞清楚它到底提供了什么。

## 它解决的核心问题

MCP 本质是一层 JSON-RPC 协议，架构是 Host（你的 Agent）→ Client → Server（包在某个数据源或 API 外面的独立进程）。它把"每个模型对接每个工具"的 **M×N 问题变成 M+N**：工具方写一次 Server，模型方实现一次 Client，两边即插即用。

协议内定义三类原语，理解它们的分工是用好 MCP 的前提：

- **Tools**：模型可主动调用的动作，如"查询订单"、"创建 issue"；
- **Resources**：应用侧控制的只读数据，如文件、配置内容；
- **Prompts**：预置的提示词模板。

## 最小可用路径

1. 用官方 SDK（Python 推荐 FastMCP）写一个最简 Server，暴露一个只读工具，比如 `get_weather(city)`；
2. 用 **MCP Inspector 单独调试**，确认工具列表、入参 schema、返回格式正常——这步别跳过；
3. 接入客户端（Claude Desktop，或在你的 Agent 框架里挂 MCP client），本地走 stdio 传输，远程服务走 Streamable HTTP；
4. 观察模型实际怎么选工具、传什么参数，根据真实调用记录迭代工具描述。

## 踩坑点

- **描述写不清，模型就选错**。模型完全靠 name + description 决定调不调、怎么调。要像给实习生写接口文档一样说清参数含义、单位、适用场景。
- **工具数量失控**。有人把一个 REST API 的五十个端点全包成工具，上下文直接膨胀，选择准确率也崩。宁可合并成几个粗粒度工具。
- **stdio 环境变量问题**。本地 Server 由客户端拉起进程，依赖的环境变量、路径、Python 版本要显式写进客户端配置，别假设继承你的 shell 环境。
- **写操作没有确认流**。Tools 自带副作用，删除类、支付类工具不给一层人工确认，出事只是时间问题。
- **提示注入**。工具返回内容会进入模型上下文，网页或邮件里的一句话可能诱导模型调用危险工具，高权限工具要做白名单。
- **SDK 版本漂移**。协议还在演进（传输层已从纯 SSE 迁到 Streamable HTTP），Client 和 Server 的 SDK 要配套升级。

## 可复用建议

- 先只读后写入，写工具默认加确认环节；
- 每次调用记录参数与返回，排障基本全靠这份日志；
- Server 独立测试通过再接 Agent，不要在 Agent 里调协议层 bug；
- GitHub、文件系统、数据库这类通用需求优先复用社区现成 Server，精力留给自己的业务工具。

## 总结

MCP 没有解决 Agent 的智能问题，它解决的是工程问题：统一的工具发现、调用与上下文投递标准。协议本身一小时能学会，真正的工作量在工具粒度设计、权限控制和上下文预算管理上——这些环节过去写插件时省不掉，只是现在有了统一的位置去做。对维护多 Agent、多插件的团队来说，这笔标准化的时间投入是划算的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/563ec5258c09c1ad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/85138590f8c715b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/1d56fc88f4b8fcbc.png)

