---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 39120
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

Agent 要干活，光会聊天不够，得能查数据库、调 API、读本地文件。2024 年 11 月 Anthropic 开源了 Model Context Protocol（MCP），一年多下来它基本成了 Agent 接外部能力的事实标准——桌面助手、IDE 插件、各类 Agent 框架都在接，OpenClaw 里做自动化插件时也绕不开它。

## 它解决的是什么问题

一句话：把 Agent 接外部能力的 **M×N 问题压成 M+N**。

MCP 之前，每个 Agent 应用要接每类工具/数据源，都得写一次私有胶水代码：定义 schema、处理鉴权、拼上下文。5 个应用 × 8 个工具 = 40 份集成代码，互不通用。

MCP 基于 JSON-RPC 2.0 定义了一层标准接口，核心是三类原语：

- **Tools**：模型可调用的动作（发请求、写文件）
- **Resources**：应用侧控制的只读上下文（文档、配置）
- **Prompts**：用户触发的提示模板

工具方写一次 MCP server，所有兼容 host 都能用；应用方实现一次 MCP client，接任意 server。传输层约定了 stdio（本地进程）和 Streamable HTTP（远程）两种方式，能力发现、版本协商、鉴权也都有规范。

## 最小可跑路径

以 Python 官方 SDK 为例，半小时能跑通：

1. `pip install "mcp[cli]"`，用 FastMCP 写 server，先只暴露一个 tool：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def get_ticket(id: str) -> str:
    """按工单 ID 查询工单状态，输入形如 T-1234。"""
    return query_db(id)

mcp.run(transport="stdio")
```

2. 用官方 Inspector 自测：`mcp dev server.py`，确认 tool 列表、参数 schema、返回内容正常。
3. 在 host 侧注册（桌面助手的 json 配置，或 OpenClaw 的插件配置里填 command/args），重启后确认 tool 被加载。
4. 真实调用几轮：核对模型填参是否正确、失败时错误信息能否合理回传给模型自行纠错。

## 踩坑点

- **stdout 污染**：stdio 模式下 JSON-RPC 走 stdout，一行 `print()` 调试输出就能把协议流打崩。日志一律走 stderr。
- **工具描述就是接口**：模型只看 docstring 和 schema 决定调不调、怎么调。写"查询数据"这种描述必然误调用，要写清输入格式和适用场景。
- **工具数量失控**：单 server 挂 40 个 tool，上下文占用和选择错误率都会恶化。按业务域拆 server，同质工具做合并。
- **Resources 和 Tools 用混**：只读数据别包成 tool，否则模型会"调用"一个本来只需要读取的东西。
- **把 server 输出当可信输入**：tool 返回内容会进入模型上下文，恶意网页或文件可能借道注入指令。shell、写文件这类高危工具务必加确认机制或白名单。

## 可复用建议

- 先只用 Tools 跑通闭环，Resources / Prompts 后置，不要一上来追求全家桶；
- 一个业务域一个 server，像对待普通 API 一样做版本管理和明确错误码；
- 远程部署用 Streamable HTTP，鉴权放网关层，别让 server 裸奔公网；
- 调试优先用 Inspector 复现，不要在 host 里盲猜。

## 总结

MCP 没有让模型变聪明，它只是把"接线"标准化了——更像 USB-C：本身不提供能力，但让能力可插拔。对个人，一次编写到处接入；对团队，工具层和应用层可以分头演进。如果你在做 Agent 自动化，建议先花半小时跑通上面的最小例子，用真实手感判断要不要深入，而不是先读完整份协议规范。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/cce0d2321d53c05d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/fa0ab3c5bd4cd549.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/ddcd9270fc17cdf7.png)

