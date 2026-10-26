---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42612
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

如果你同时维护过几个 Agent 项目，大概率写过类似的胶水代码：这个项目里封装了一遍飞书 API，那个项目里重写了 GitHub 调用，第三个又自己实现了一遍数据库查询。模型能力迭代很快，但“让模型安全地拿到外部上下文”这一层，每个项目都在重复造轮子。

MCP（Model Context Protocol）就是针对这一层的开放协议，2024 年底开源，线上走 JSON-RPC 2.0，把"Agent 应用 ↔ 外部工具/数据"的接口标准化了。

## 它到底解决了什么

本质上是一个积分问题。没有 MCP 时，M 个 Agent 应用接 N 个工具，最坏要写 M×N 套集成；换框架或工具升版本，集成代码跟着改。有了 MCP，应用侧实现一次客户端，工具侧实现一次服务端，复杂度降到 M+N。

对 OpenClaw 这类插件化 Agent 场景，这个差异很实际：插件作者只需按 MCP 写一个 Server，不用关心宿主是什么框架；宿主升级也不会砸掉已有插件。

## 核心概念（三分钟版）

- **Host / Client**：Agent 应用本体，内嵌 MCP Client 负责建连
- **Server**：暴露能力的进程，可提供三类东西
  - Tools：模型可主动调用的动作（发消息、查库、建 issue）
  - Resources：应用侧读取的数据（文件、配置、查询结果）
  - Prompts：预设提示词模板
- **传输层**：本地常用 stdio，远程用 Streamable HTTP

## 最小上手路径

1. 选官方 SDK（Python / TypeScript 都够用），用 FastMCP 这类封装几十行就能起一个 Server
2. 先写一个只读工具练手，比如“查询某张表的结构”，重点是写好描述——它会直接进模型上下文
3. 用 MCP Inspector 本地调试，确认工具列表和调用链路正常
4. 接入宿主（OpenClaw 或其他支持 MCP 的 Agent），配一条 stdio 命令或一个 HTTP 端点
5. 观察模型实际选工具的行为，回头修描述

## 踩坑点

- **描述敷衍，模型就乱选或不选工具**。工具名 + 参数说明就是给模型看的 API 文档，值得花时间打磨
- **一个 Server 塞几十个工具**，选择准确率明显下降。按领域拆分，单个 Server 工具数控制在个位数到十几个
- **stdio 模式下 Server 是宿主的子进程**，异常退出容易留僵尸进程，清理逻辑要写
- **工具报错别直接抛异常断连**，返回结构化错误信息，给模型自我纠正的机会
- **安全别忽视**：第三方 Server 的返回结果会进模型上下文，等于引入了注入面；来源不明的 Server 不要给写权限

## 可复用建议

- Server 做薄：只做协议转换，业务逻辑留在原有 API 里，方便独立演进
- 描述模板化：一句话讲清“做什么”，再补参数约束和失败场景
- 权限最小化：读工具默认开，写工具加人工确认
- 协议仍在演进（传输层、鉴权都在更新），锁住 SDK 小版本，升级前用 Inspector 跑一遍回归

## 总结

MCP 没有让模型变聪明，它解决的是工程问题：把上下文接入从 N 份私有胶水代码，收敛成一套标准协议。对写 Agent 的人来说，它是插件生态的地基——值得花一个下午用官方 SDK 写个最小 Server 跑通全链路，之后所有的接入和拆分决策都会清晰很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/936d64b323da6ae7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/ba961fabee94e999.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/44eb84e8af0bc2a0.png)

