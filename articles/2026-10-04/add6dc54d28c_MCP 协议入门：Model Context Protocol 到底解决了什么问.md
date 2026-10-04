---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 40405
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

做 Agent 的同学大概都有类似经历：模型能力上来了，真正卡进度的是“接工具”。每个框架都有自己的插件格式，各家的桌面端、IDE、自研 Runtime 互不通用。你给 A 框架写的查询插件，搬到 B 框架要重写一遍 schema、重接一遍鉴权。M 个应用 × N 个工具，就是 M×N 份胶水代码。

MCP（Model Context Protocol）是 Anthropic 2024 年底开源的协议，一句话概括：把 M×N 压成 M+N——工具方实现一次 MCP Server，任何支持 MCP 的宿主（Host）都能直接用。

## 它到底解决什么问题

拆开看是三件事：

1. **统一接口**。底层是 JSON-RPC 2.0，本地走 stdio，远程走 Streamable HTTP。工具的定义、调用、返回格式有了标准，框架层不用再造轮子。
2. **动态能力发现**。客户端连接后通过 `tools/list` 拿到能力清单，宿主不用硬编码工具表。服务端新增工具，下游不用发版。
3. **职责分离**。工具逻辑、鉴权、数据访问收拢在 Server 进程里，可独立升级，Python 或 TypeScript 写都行，Host 只管编排。

三个核心原语要分清：**Tools**（模型决定何时调用）、**Resources**（应用侧读取的上下文数据）、**Prompts**（用户触发的模板）。多数场景你只需要 Tools。

## 上手步骤

1. 选 SDK：官方有 TypeScript 和 Python 两套，Python 用 FastMCH 风格十几行就能起一个 stdio Server。
2. 定义工具：写清楚函数签名和描述——schema 和 description 会原样进模型上下文，这段“给模型看的文档”比代码本身更影响调用准确率。
3. 本地调试用 stdio，先拿官方 MCP Inspector 跑通 `tools/list` 和 `tools/call`，再接宿主。
4. 接入 Agent：在 OpenClaw 的 MCP 配置里注册 server，确认工具被正确枚举、权限确认行为符合预期。
5. 需要远程共享时再上 Streamable HTTP，会话管理和鉴权放网关层，别塞进业务代码。

## 踩坑点

- **stdio 模式下 print 会毁掉一切**。stdout 被 JSON-RPC 占用，任何 print 调试输出都会让宿主解析失败。日志一律走 stderr。
- **工具不是越多越好**。挂二十个 server、上百个工具，光描述就吃掉几千 token，模型选择准确率反而下降。按任务裁剪连接。
- **工具结果是不可信输入**。网页抓取、文件内容里可能藏注入指令，别给工具开 auto-approve，服务端要做参数校验。
- **描述含糊等于误调用**。`process_data` 这种名字模型猜不透；把参数单位、返回结构写进 description，误触发率会明显下降。
- **SDK 版本要配对**。协议在演进（从 SSE 迁到 Streamable HTTP 就是一次），两端版本差太多会握手失败。

## 可复用建议

- 一个 Server 对应一个领域或数据源，工具收敛到 5–8 个；宁可提供 `search_issues` 这类聚合接口，也别把 CRUD 全摊开。
- 把工具描述当 prompt 写，Server 做版本化。
- 尽量无状态；长任务用进度通知，不要阻塞请求。
- 团队内沉淀通用 Server（数据库、内部 API、CI），比每个项目各写一套插件划算得多。

## 总结

MCP 没有让工具变得“更聪明”，它只是把集成方式标准化了——而在 Agent 工程里，标准化恰恰是复利最高的那类基建。如果你的场景只是“一个 Agent 接三五个自有工具”，手写 function calling 也许够用；一旦工具来源开始跨团队、跨项目，MCP 的迁移成本远低于继续堆胶水代码。建议从一个最小 stdio Server 起步，用 Inspector 跑通全链路，再决定要不要上远程部署。欢迎在评论区贴出你们第一个跑通的 MCP Server。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/12fcd2d51b8f04cf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/1870492bc351b94f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/0bf156e1d66b0824.png)

