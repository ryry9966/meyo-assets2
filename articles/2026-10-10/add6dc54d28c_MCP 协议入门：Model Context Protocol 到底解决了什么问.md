---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41117
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

做 Agent 绕不开一件事：让模型能调用真实世界的能力——读文件、查数据库、发消息、调 API。在 MCP 出现之前，这件事没有统一约定：每个 Agent 框架自定义一套 function calling 格式，每个工具提供方各自封装接入代码。最坏情况下，M 个客户端乘 N 个工具，要写 M×N 份胶水；你给 A 框架写的查库插件，换到 B 框架，schema、鉴权、错误处理全部重来。MCP（Model Context Protocol）要解决的，就是这一层集成问题。

## 它具体解决了什么

一句话：把「Agent 如何发现、描述并调用外部工具」标准化成一个开放协议。

MCP 是 Host–Server 架构：Agent 作为 Host 内嵌 MCP Client，工具以独立 Server 进程存在，双方用 JSON-RPC 2.0 通信——本地走 stdio，远程走 Streamable HTTP。Server 对外暴露三类原语：

- **Tools**：模型可主动调用的动作，偏执行；
- **Resources**：可读取的数据，偏只读；
- **Prompts**：预置提示模板，由用户触发。

初始化握手时双方协商能力，之后 Client 通过 `tools/list` 拿到工具清单和参数 schema，用 `tools/call` 发起调用。对工具作者，写一次 Server，任何支持 MCP 的宿主都能接；对 Agent 用户，装一个 Server 就多一类能力，不用改 Host 代码。

## 在 OpenClaw 中接入一个 MCP Server

1. **选传输方式**：本地工具优先 stdio，一条命令能拉起的用 `npx`；跨机器或需常驻的服务用 Streamable HTTP。
2. **注册 Server**：在 OpenClaw 的 MCP 配置段声明 `command`、`args`、`env`，API key 之类的敏感信息通过环境变量注入。
3. **验证发现**：重启后在单独会话里让 Agent 枚举工具，确认 `tools/list` 返回的清单和参数描述符合预期。
4. **小任务试跑**：先跑一个只读操作（如列出记录），观察模型填参是否正确、报错能否被理解。
5. **收敛权限**：只启用需要的工具，写操作默认人工确认，之后再挂进自动化流程。

## 踩坑点

- **stdout 污染**：stdio 模式下 stdout 只能承载 JSON-RPC，Server 里任何 `print` 调试输出都会打断协议流，日志一律走 stderr。新手第一大坑。
- **description 是给模型看的 prompt，不是注释**：写得含糊，模型就选错工具、填错参数。用模型视角描述：什么场景用、参数含义、返回什么。
- **工具数量失控**：几十份 schema 全塞进上下文，又贵又降低选择准确率，按会话目的启用子集。
- **Windows 下的 stdio**：直接拉 `npx` 常见 shell 包装问题，command 写法要适配。
- **安全边界**：装第三方 Server 等于把一段本地能力交给模型驱动，文件路径、网络访问、鉴权范围先审一遍再用。
- **版本演进**：早期基于 SSE 的教程已过时，参考实现与 spec 版本要对齐，配置里锁死版本。

## 可复用建议

- 把散落的自动化脚本收拢成 MCP Server，一个 Server 管一类资源（比如全部围绕本地笔记），工具控制在个位数、彼此正交。
- 返回值结构化且可截断：大结果提供摘要和分页参数，别让一次调用撑爆上下文。
- 新 Server 先在交互会话里验证，再进定时任务。排查时用「枚举工具 → 手动调用一次」二分定位，区分是发现层还是执行层的问题。

## 总结

MCP 不是让模型变聪明的魔法，它解决的是纯工程问题：把 M×N 的集成成本压成 M+N。对 OpenClaw 用户来说，最实际的收益是工具变成了可迁移资产——今天写好的 Server，换任何支持 MCP 的宿主都能继续用。如果你手里已经有几个只为某个 Agent 服务的脚本，把它们改成 MCP Server，就是理解这套协议最快的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/e8c7f7eaecfbab96.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/6631bd0c9e78e098.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/86773a028b8fc9d1.png)

