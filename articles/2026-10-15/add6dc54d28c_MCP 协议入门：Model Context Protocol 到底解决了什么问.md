---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41588
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景：工具接入的 M×N 困境

做 Agent 的同学大多经历过这种场景：手上有三四个框架，每个框架接数据库、文件系统、内部 API 的方式都不一样。同样是"查一下订单表"，在框架 A 里要写一套 function calling 的 schema，在框架 B 里要实现它的插件接口，在框架 C 里可能得塞进自定义 prompt 模板。应用越多、工具越多，集成成本就是 M×N 次重复劳动。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），思路很直接：把"应用怎么调用工具"这件事标准化。应用侧实现一次 MCP 客户端，工具侧实现一次 MCP 服务端，M×N 就变成了 M+N。

## 问题：MCP 到底规范了什么

剥开宣传，MCP 实际只解决三件事：

1. **发现**：客户端连上服务端后，通过 `tools/list` 拿到有哪些工具、参数 schema 是什么，Agent 不需要硬编码工具清单；
2. **调用**：统一走 JSON-RPC 2.0 的 `tools/call`，输入输出有明确的序列化约定；
3. **上下文注入**：除工具外，还有 resources（文件、库表结构等只读数据）和 prompts（可复用提示词模板）两类原语，"给模型喂什么"也有标准姿势。

传输层主流是两种：stdio（本地子进程，个人使用最简单）和 Streamable HTTP（远程共享服务）。早年的 SSE 传输已标记废弃，选型时注意。

## 做法：写一个最小 MCP Server

以 Python 官方 SDK 为例，几行就能跑起来：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("order-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询状态，返回订单状态和最近一条物流记录。"""
    return f"order {order_id}: shipped"

mcp.run()  # 默认 stdio 传输
```

接入步骤：

1. 在宿主（Claude Desktop、OpenClaw 的 MCP 客户端配置等）里注册 server 启动命令；
2. 先用官方 MCP Inspector（`npx @modelcontextprotocol/inspector`）独立调试，确认工具列表和返回结构没问题；
3. 再挂进 Agent 流程，观察模型是否按预期选工具。

## 踩坑点

- **描述写不清，模型就选错**。模型选工具完全依赖名称和 description，按"给新实习生看的 API 文档"标准写，讲清输入输出和适用边界。
- **工具堆太多，选择率反而下降**。单 server 挂几十个工具会稀释上下文，按场景拆分 server，宿主侧做工具过滤。
- **大返回值撑爆上下文**。读整个文件、返回整张表都是常见事故，server 侧做分页截断，超长内容落盘后只给路径。
- **stdio 的阻塞问题**。server 是宿主的子进程，一个同步阻塞调用会卡住整条链路，慢操作务必异步或设超时；Windows 下还要注意启动命令的包装差异。
- **版本兼容**。协议演进很快，旧 client 配新 server 偶尔会静默失败，排查时先确认双方协议版本。

## 可复用建议

- 一个 server 只管一个领域，工具面尽量小；
- 错误信息作为工具结果返回，而不是直接抛异常，给模型自我修正的机会；
- 业务编排放在 Agent 侧，MCP server 只做薄适配层，别把工作流塞进 server；
- 本地自用走 stdio，团队共享能力封装成 Streamable HTTP 服务并配好鉴权。

## 总结

MCP 不是让 Agent 变聪明的魔法，它只是把"工具接入"从私人定制变成了标准接口，真正的价值在生态复用：你写的 server 能被任何兼容宿主直接挂载。对个人开发者，从 stdio + 两三个工具起步最划算；对团队，优先把高频数据源（数据库、知识库、工单系统）各封一个 server，收益最明显。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/43d563090eba530a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/4864d658f0c7d1de.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/423747bee489e47c.png)

