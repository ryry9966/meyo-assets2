---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42870
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

Agent 要做实事，绕不开两件事：调用工具、读取上下文。在 MCP 出现之前，这件事没有统一做法——各家 IDE 有自己的扩展体系，不同 Agent 框架靠手写 function calling 胶水代码，同一个内部 API 可能要为不同宿主各写一遍适配器。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标是把「模型如何获取上下文、如何调用工具」标准化。如今主流 IDE、Agent 框架和大量自建 Agent 都已支持，可以当作事实标准来用。

## 它解决了什么问题

核心是 **M×N 集成问题**：M 个模型/宿主 × N 个数据源，过去要写 M×N 份胶水代码。MCP 把它拆成 M+N：宿主实现一次 client，工具方实现一次 server，中间用统一协议握手。

具体拆开，MCP 定义了三类能力：

- **Tools**：模型可主动调用的函数，即 tool use；
- **Resources**：宿主应用控制的数据（文件、表结构、日志），供注入上下文；
- **Prompts**：用户侧可选择的模板。

传输层主要是两种：本地进程用 stdio，远程服务用 Streamable HTTP。报文格式是 JSON-RPC 2.0，连接建立时先做能力协商。

一句话：MCP 不会让模型变聪明，它把「接入」从一次性定制工作变成了标准工程问题。

## 上手：写一个最小 server

以本地场景为例，用 Python SDK 十几行就能跑通：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回 JSON 字符串"""
    return db.lookup(order_id)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

步骤：

1. **选一个场景**：先接一个内部系统（订单库、Wiki、监控），别一上来接十个；
2. **选传输**：本地调试用 stdio，跨机部署再上 Streamable HTTP；
3. **写 server**，每个工具写清楚 docstring——那就是给模型看的 API 文档；
4. **在宿主配置里注册** server 启动命令（Claude Desktop、IDE 或你自己的 OpenClaw Agent 均可）；
5. 用官方 **MCP Inspector** 连一遍，验证工具列表、试调用、看原始报文。

## 踩坑点

- **日志打到 stdout 会让协议挂掉**。stdio 传输下 stdout 就是协议通道，`print` 调试会直接破坏 JSON-RPC 报文。日志一律走 stderr。
- **工具描述含糊，模型就乱选**。`get_data(id)` 这种签名模型猜不出语义，参数约束、返回示例、失败条件都要写进描述。
- **工具数量失控**。一个 server 注册四五十个工具，上下文膨胀且选择准确率下降，按域拆分 server，单个保持小而精。
- **权限默认最小化**。server 是信任边界，别把整个文件系统或可写数据库句柄直接暴露，先只读、限路径。
- **SDK 版本与协议修订要对齐**，升级宿主后若握手失败，先查 initialize 阶段的版本协商日志。

## 可复用建议

- 优先「包一层」：已有 REST API 用 MCP server 包装，业务逻辑不必重写；
- 工具设计成幂等的——模型重试是常态，不是异常；
- 长耗时任务返回任务 ID 并提供查询进度的工具，别让一次调用挂住整个会话；
- 把工具描述当接口文档维护，纳入 code review。

## 总结

MCP 的价值不在技术多前沿，而在它把生态碎片收敛了：一套协议覆盖工具、资源、模板三类上下文，宿主和数据源各自实现一次即可互通。对做自动化和 Agent 的团队来说，新接一个内部系统从「写适配器」变成「起一个 server」——这才是它真正的工程意义。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/0be4089acece2781.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/267a889218c097ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/5043bf07be657167.png)

