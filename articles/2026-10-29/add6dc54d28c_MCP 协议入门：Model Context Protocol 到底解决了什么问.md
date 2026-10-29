---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42937
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景：工具调用一直有，但"各说各话"

Function calling 出现之后，模型确实能调工具了，但接入层长期是碎片化的：每个客户端都自定义插件格式，自研 Agent 干脆手写 glue code。工具本身不难写，难的是每换一个宿主就要重写一遍接入。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），基于 JSON-RPC 2.0，把"Agent 怎么发现工具、怎么调用、怎么拿数据"标准化。一年多下来，主流客户端和 SDK 基本都跟进了，它正在变成 Agent 生态里的"USB-C 接口"。

## 问题：M×N 的接入矩阵

没有统一协议时，M 个客户端要接 N 个工具/数据源，最坏要写 M×N 份适配代码。MCP 把它压成 M+N：工具方写一次 server，客户端实现一次协议，两边即插即用。

对个人 Agent 用户的收益更直接：同一批 MCP server（文件系统、浏览器、数据库、内部 API）可以在不同宿主间复用，不用为每个 Agent 重新封装。

## 做法与步骤

三步跑通：

1. **装现成 server 试水**：官方维护了 filesystem、fetch、git 等常用 server，stdio 传输，配置即用，先建立体感。
2. **用 Inspector 调试**：`npx @modelcontextprotocol/inspector` 可以不接模型直接列出 tools/resources，手动调用验证返回结构，这是排障第一入口。
3. **接进客户端**：在 OpenClaw 或其他宿主的 MCP 配置里声明 server 命令与参数，重启后确认工具列表被拉起，再让它执行一个最小任务（如"列出某目录的文件"）验证闭环。

自己写 server 用官方 SDK（Python/TypeScript），核心是把函数注册成 tool 并写好 description，协议层 SDK 已经处理掉了。

## 踩坑点

- **description 是写给模型看的**，不是写给人的文档。参数含义、返回结构、何时该用/不该用，都要讲清楚，否则模型选工具和传参基本靠猜。
- **工具数量要克制**。挂几十个工具会明显污染上下文，模型选错的概率上升。按域拆 server、按需启用。
- **stdio 模式下 stdout 就是协议通道**，任何 `print` 调试输出都会破坏 JSON-RPC 流，调试信息必须走 stderr。
- **工具返回值直接进入模型上下文**，等于引入了二次注入面。不可信来源的 server 做白名单和最小权限，别一上来就给文件写入和 shell 执行。
- **server 升级后工具签名变化**，客户端缓存的旧描述会导致调用失败，改 schema 要同步重启客户端。

## 可复用建议

- 一个 server 只管一个域：文件、数据库、搜索分开写，粒度细才好复用和授权。
- 优先做"只读工具"，写操作单独拆出去，方便宿主侧加确认和审批环节。
- 把 Inspector 里的手动调用记录当回归用例，改完 server 先过一遍再发布。
- 工具设计偏"粗粒度动词"：一个 `search_and_summarize` 往往比让模型自己串三个原子工具更稳。

## 总结

MCP 没有发明新魔法，它做的事是把工具接入从"每家一套私货"收敛成一个开放协议。对 Agent 实践者，价值在于：工具写一次、到处能用，且发现、调用、权限有了统一语义。先跑通官方 server，再按域拆自己的 server，控制工具数量、认真写描述，就能把它稳定地用进自动化流程里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/a18bed818d2694ce.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/2d2365b5742e67c2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/d671ab8ab0e16aa0.png)

