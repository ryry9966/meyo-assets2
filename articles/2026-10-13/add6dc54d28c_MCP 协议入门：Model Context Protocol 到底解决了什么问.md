---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41371
source: 综合讨论
publishedAt: 2026-10-13
---

# 背景

做 Agent 的人大概率都遇到过同一个困境：模型的推理能力不缺，缺的是"够得着"的上下文和工具。想让 Agent 读本地文件、查数据库、调内部 API，每接一个数据源就得写一套胶水代码，换个框架再来一遍。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），定位很朴素：给"模型应用 ↔ 工具/数据源"之间定一个开放协议，思路类似 LSP 之于编辑器。OpenClaw 生态里 MCP 已经是插件接入的主流方式之一，值得把底层逻辑搞清楚。

# 问题：M×N 集成爆炸

没有标准时：

- 有 M 个 AI 应用、N 个工具，最坏要维护 M×N 套适配；
- 每个工具的鉴权方式、参数格式、错误语义各不相同；
- 同一个能力（比如查 Postgres）在每个 Agent 框架里重复实现；
- 上下文注入（文件、日志、检索结果）也没有统一入口。

MCP 的价值是把 M×N 压成 M+N：工具方写一次 MCP Server，任何支持协议的宿主都能直接用。

# 协议最小集合

- 基于 JSON-RPC 2.0；
- 三个核心原语：**Tools**（模型可调用的函数，带 JSON Schema 参数）、**Resources**（可读取的上下文数据）、**Prompts**（预置提示词模板）；
- 两种传输：stdio（本地子进程）和 Streamable HTTP（远程服务）。

# 做法：接一个最小 MCP Server

1. 选一个真实的小场景，比如让 Agent 查询内部服务状态。
2. 用官方 SDK（Python/TypeScript 均可）定义工具：

```python
@mcp.tool()
def get_service_status(service: str) -> str:
    """查询指定服务的运行状态，返回 JSON 字符串"""
    ...
```

3. 在宿主配置里注册该 server，本地调试用 stdio 最省事。
4. 跑通一次完整调用，重点观察三件事：模型是否选对了工具、参数是否合法、返回是否撑爆上下文。

# 踩坑点

1. **工具描述就是 prompt。** 模型靠 description 决定用不用这个工具，写"查询数据"这种模糊描述，模型会在不该调的时候乱调。
2. **工具不是越多越好。** 单个 server 暴露 20+ 个工具，选择准确率明显下降。按领域拆，一个 server 只管一件事。
3. **输出要节流。** 返回几万字符的日志直接进上下文，token 烧得快还干扰推理。默认截断 + 提供分页参数。
4. **安全边界。** MCP Server 拿的是你的本机凭证和环境变量，第三方 server 不要盲装；工具返回内容里可能藏注入指令，所有写操作务必加人工确认。
5. **stdio 与 HTTP 别混着配。** stdio server 的 cwd、环境变量继承经常出问题，报"工具不存在"先查子进程有没有起来。

# 可复用建议

- 起步只做 2~3 个工具，先跑通闭环再扩展；
- description 里写清三件事：干什么、参数含义、失败时返回什么；
- 工具命名加命名空间前缀（如 `github_create_issue`），避免多 server 重名；
- 每次调用记日志：入参、耗时、返回大小，排障基本靠这个；
- 读操作和写操作分开实现，写操作默认要确认。

# 总结

MCP 解决的不是模型能力问题，而是"接入"这个脏活的标准化问题。它把各家私有的胶水代码换成一个开放协议，价值核心在组合性：工具生态每多一个 server，你的 Agent 就多一批能力，而宿主侧一行代码不用改。它并不神秘，本质是给模型装了一个带规范的通用接口。建议从身边一个真实的小需求入手写一个 server，比读十篇介绍文章有用得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/6779d10be57796dd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/efa164c16f1053d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/f32caf680ef88467.png)

