---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38080
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

把模型接到真实世界，一直是 Agent 落地里最繁琐的一环：读文件、查数据库、调内部 API、发消息，每个工具都要为每个客户端单独写一遍胶水代码。N 个工具 × M 个宿主应用，就是 N×M 套适配层，且互不通用。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），把「模型应用 ↔ 外部工具/数据」这一层的接口标准化。它本质上是一个基于 JSON-RPC 2.0 的协议，不是框架也不是产品——可以理解成工具生态里的 USB-C 接口。

## 它到底解决什么问题

一句话：把 N×M 的集成问题降成 N+M。

- 工具方只需实现一次 MCP Server，对外暴露三类能力：Tools（模型可调用的动作）、Resources（可读取的数据）、Prompts（预置提示模板）；
- 宿主应用（Agent、IDE、桌面客户端）只需实现一次 MCP Client；
- 传输层支持 stdio（本地子进程）和 Streamable HTTP（远程服务），上层协议一致。

对 OpenClaw 用户来说，价值在于：插件能力从「写死在某一代码库里」变成独立进程、可组合、可复用，Agent 主程序和工具实现彻底解耦。

## 最小可用路径

以 Python SDK 为例，四步：

1. 安装官方 SDK（`pip install "mcp[cli]"`）；
2. 写一个约 20 行的 server：用装饰器注册一两个 tool，认真写函数 docstring——它会被原样放进模型上下文；
3. 在宿主配置里登记该 server 的启动命令（stdio 模式就是一条命令行），重启会话；
4. 先用官方 MCP Inspector（`mcp dev xxx.py`）脱离宿主单独调试，确认工具列表、入参 schema、返回结构符合预期，再接入 Agent。

## 踩坑点

- **工具描述就是 prompt。** docstring 含糊，模型要么不调用、要么传错参数。写清楚：什么场景该用、参数单位、失败时返回什么。
- **工具数量失控。** 一个 server 挂 40 个功能重叠的工具，模型选择准确率明显下降。按领域拆 server，单 server 控制在 10 个以内。
- **stdio 静默失败。** server 崩了往往只表现为「工具列表为空」，先看 stderr 日志，再查 Python 版本和依赖冲突。
- **返回值别贪大。** 把整张表丢回上下文，几轮就把窗口撑爆。做分页和字段裁剪，「给模型的摘要」和「原始数据」分开返回。
- **权限即风险。** MCP server 以你的身份运行，工具返回内容会被模型当作可信上下文——第三方 server 存在提示注入面，删除、支付类操作务必加人工确认或最小权限。

## 可复用建议

- 把团队常用 server 的配置收进一个 git 仓库统一管理，成员一键拉起；
- 一个 server 对应一个领域（搜索、数据库、工单），别做万能大杂烩；
- 错误信息写给模型看：返回结构化、可执行的提示（如「日期格式应为 YYYY-MM-DD」），比抛异常更能触发自愈；
- 升级 SDK 后先跑一遍 Inspector 回归——协议仍在演进，传输层就从 HTTP+SSE 迁移到了 Streamable HTTP。

## 总结

MCP 解决的不是「模型更聪明」，而是「生态接口不统一」这个纯工程问题。它换来的是：工具写一次、处处可接，插件独立演进，Agent 主程序保持瘦内核。对做自动化和插件的团队，现在就值得把内部工具逐步 MCP 化——迁移成本不高，复利很明显。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/ee0ef8760d98ca7d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/fde9b113058c57c8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/63f71664ef8353d7.png)

