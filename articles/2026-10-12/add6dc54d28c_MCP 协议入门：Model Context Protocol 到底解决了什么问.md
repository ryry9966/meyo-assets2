---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41311
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

模型本身只会生成文本，它看不到你本地的代码库、数据库和内部 API。2023 到 2024 年，各家 Agent 框架都在解决同一个问题：怎么把工具"塞"给模型。结果就是每套框架发明一套自己的工具描述格式、调用约定和鉴权方式——桌面端 AI 助手、IDE 插件、自建 Agent 各自为政，同一个能力的接入逻辑要重复写好几遍。

## MCP 到底解决了什么

MCP（Model Context Protocol，Anthropic 2024 年 11 月开源）本质上是把"工具接入"做成了标准协议：

- **传输层**：JSON-RPC 2.0。本地用 stdio，远程用基于 HTTP 的流式传输。
- **架构**：Host（Agent 客户端，比如 OpenClaw 这类运行时）内置 Client，与若干 Server 通信；Server 是独立进程，负责暴露能力。
- **三类原语**：tools（模型可调用）、resources（可读取的上下文数据）、prompts（预置模板）。绝大多数自动化场景只用 tools。

一句话总结：把 **M×N 的集成问题变成 M+N**。工具方实现一次 Server，任何支持 MCP 的 Host 都能直接用。

## 最小可运行示例

用官方 Python SDK 写一个只读 Server，二十行以内：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("fs-tools")

@mcp.tool()
def list_files(path: str) -> list[str]:
    """列出指定目录下的文件名，只读，不递归。"""
    import os
    return os.listdir(path)

mcp.run()
```

在 Host 的 MCP 配置里以 stdio 方式注册这个脚本，重启后模型就能看到 `list_files` 这个工具。接入成本主要在配置，不在代码。

## 踩坑点

1. **工具描述是给模型看的 prompt，不是给开发者看的注释。** 写"查询数据"这种模糊描述，模型分不清它和别的工具的边界，会选错工具或乱传参数。参数格式、返回结构、适用场景都要写清楚。
2. **返回值直接吃 context。** 一个返回几千行 JSON 的工具，一次调用就能把上下文窗口撑爆。做分页、截断，只返回必要字段，必要时返回 count 加摘要。
3. **stdio 子进程生命周期。** Host 退出时 Server 不一定干净退出，调试期间经常留僵尸进程占着端口或文件锁。开发期建议显式管理进程。
4. **第三方 Server 是攻击面。** 工具返回的内容会进入模型上下文，存在间接注入风险；Server 本身还拿着你的凭据。只装可信来源，先只读，敏感目录不要挂上去。
5. **远程 Server 的鉴权比 stdio 麻烦得多。** 本地调试一切正常，一换 HTTP 传输就要面对 OAuth 和 token 刷新，请预留额外时间。

## 可复用建议

- 从只读工具起步；所有写操作加确认步骤或 dry-run 模式。
- 工具颗粒度切小：宁要 `read_file` / `grep` 的组合，不要一个万能 `do_everything`。
- 每次工具调用记日志（入参、出参大小、耗时），排查 Agent 的奇怪行为基本全靠它。
- 评估用真实 trace：让 Agent 跑十几次真实任务，观察它是否选对工具、传对参数，这比单元测试更能暴露描述层面的问题。

## 总结

MCP 解决的是接口标准化问题，不是模型能力问题。它值得投入的时机很明确：你要接的工具超过三四个，或者同一套工具需要服务多个 Host / 多个 Agent。如果只是单场景的一次性脚本，直接写函数调用反而更省事。

工程判断标准就一条：**当重复造轮子的成本开始超过学协议的成本时，就该上 MCP 了。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/f1044d34421a7fef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/5df1bdc4c46b7b51.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/b5c203117e01a029.png)

