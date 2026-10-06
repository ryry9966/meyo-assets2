---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 40726
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），如今它基本成了 Agent 接外部工具和数据的事实标准。不少新同学第一次接触会问：function calling 不是已经能调工具了吗？MCP 到底解决了什么？

## 问题：M×N 的接入税

没有 MCP 之前，接工具是纯手工业。你想在某个 Agent 里用 GitHub、数据库、浏览器，就得按那个框架的调用格式手写 JSON Schema、包一层适配、处理鉴权和重试。换一个宿主，同样的活再干一遍。工具供给方有 M 家，宿主方有 N 家，最坏情况就是 M×N 份胶水代码，而且每份都在悄悄腐化。

MCP 把这笔税砍成 **M+N**：工具方实现一次 MCP Server，声明"我有哪些工具、参数结构长什么样、结果怎么返回"；宿主方实现一次 MCP Client；中间用统一协议对接。类比 USB-C——设备厂商和电脑厂商各自遵守一个接口标准，就不再需要一对一做转接头。

## 核心模型：三层 + 三原语

- **Host / Client / Server**：宿主（如 OpenClaw、IDE、桌面助手）内嵌 Client，每个 Client 与一个 Server 保持一对一连接。
- **三类能力原语**：Tools 是模型可主动调用的动作；Resources 是宿主可读取的上下文数据；Prompts 是可复用的提示模板。日常自动化九成场景用的都是 Tools。
- **传输层**：本地用 stdio（Server 是宿主拉起的子进程），远程用 Streamable HTTP。

## 在 OpenClaw 里跑通最小闭环

1. 明确一个真实需求，比如让 Agent 查本地 SQLite、操作 Git 仓库。别一上来挂十个 Server。
2. 先复用再自研。生态里 filesystem、git、sqlite、playwright 等都有现成实现。
3. 在 OpenClaw 配置里声明 Server：stdio 方式给 command/args/env，远程方式给 url 和凭据。
4. 重启后验证：确认工具列表被正确拉起，手动触发一次最简单的调用，核对入参出参是否符合 Schema。
5. 收紧权限：工作目录限定、只读账号、高危操作（写库、执行命令）开启人工审批。

## 踩坑点

- **stdio 启动环境和终端不同**。宿主进程拉起子进程时的 PATH、Node/Python 版本经常和你的 shell 不一致，npx/uvx 找不到或解析到旧版本是高频事故；Windows 下还有 npx.cmd 的坑。
- **工具数量爆炸**。Server 挂多了，几十个工具定义塞进上下文，token 涨是小事，模型选错工具的概率会明显上升。按 Agent 的职责只挂必要的 Server。
- **工具名冲突**。两个 Server 都提供 search 很常见，注意宿主的命名空间/前缀策略。
- **信任边界**。MCP Server 是任意代码，跑在你机器上、拿着你的环境变量。装社区 Server 前读源码，API Key 按最小权限发放。
- **别混淆三原语**：Resources 不会自动注入上下文，Prompts 也不能被模型"调用"。

## 可复用建议

- 一个 Server 只覆盖一个领域，保持薄。要接内部系统，就写个二三十行的小 Server 包自己的 API，比塞进大而全的通用 Server 好维护得多。
- 给工具调用加日志，危险操作加 dry-run 参数，出问题能回放。
- 锁定 Server 版本，升级前在独立配置里先试。
- 把"写一个 MCP Server"当资产：一次实现，OpenClaw、IDE、其他宿主都能复用——这才是 M+N 的真正红利。

## 总结

MCP 没有让模型变聪明，它做的是把"接入"这件事标准化：能力用统一协议描述，集成成本从 M×N 降到 M+N。对 OpenClaw 用户来说，务实的路径是先用现成 Server 跑通一个闭环，再按需写薄 Server 沉淀自己的能力库。协议本身不复杂，真正难的是权限与信任管理——这部分没有银弹，只能靠最小权限和审批流兜住。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/71722dc0bbe9aefd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/1ababd981200b442.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/fe8be66f9f686b24.png)

