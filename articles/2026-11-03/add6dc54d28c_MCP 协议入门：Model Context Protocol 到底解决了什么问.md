---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 43313
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

做 Agent 自动化的人大多经历过同一个阶段：模型能力够了，但"接不上东西"。想让它查数据库、调内部 API、读本地文件，每接一个数据源就要写一套胶水代码；换一个宿主（IDE、桌面端、自己的 runtime）还得再写一遍。这是典型的 M×N 问题——M 个应用对 N 个数据源，组合爆炸。

MCP（Model Context Protocol）是 2024 年底开放的协议，一句话概括：把"应用怎么连接模型上下文和工具"标准化，把 M×N 降成 M+N。数据源方写一次 MCP Server，所有支持 MCP 的宿主都能用；应用方实现一次 Client，就能接入所有 Server。

## 它到底解决了什么

协议里只有三个核心角色：

- **Host**：模型运行的宿主，比如桌面客户端、IDE，或你自己写的 Agent runtime（OpenClaw 这类场景）。
- **Client**：Host 内部与 Server 保持 1:1 连接的中间层。
- **Server**：对外暴露三类能力——**Tools**（模型可调用的动作）、**Resources**（可读取的数据）、**Prompts**（可复用的提示模板）。

通信走 JSON-RPC，本地用 stdio，远程现在推荐 Streamable HTTP。握手时 Client 和 Server 做能力协商，各自声明支持什么，从协议层面避免了"插件格式不兼容"这类扯皮。

## 跑通最小链路

用官方 Python SDK，十几行就能起一个 stdio Server：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-tools")

@mcp.tool()
def get_ticket(id: str) -> str:
    """按工单 ID 查询状态，返回 JSON 字符串。"""
    return query_db(id)

mcp.run()  # 默认 stdio
```

验证环节别偷懒：不要直接接进 Agent，先用官方 Inspector（`npx @modelcontextprotocol/inspector`）跑一遍，确认工具列表、入参 schema、返回内容都符合预期，再配置到宿主。这一步能省掉大部分"模型行为怪异"的排查时间。

## 踩坑点

1. **stdout 污染**。stdio 模式下整条通道都是 JSON-RPC，任何 `print()` 调试输出都会让连接直接断掉。日志一律走 stderr。
2. **工具描述写得太省**。描述就是模型眼里的 API 文档，"查询数据"这种描述会导致模型瞎传参。写清楚参数含义、返回结构、什么场景该用这个工具。
3. **一上来挂几十个工具**。工具越多，上下文越膨胀，模型选错的概率越高。按业务域拆 Server，按需挂载。
4. **权限默认太大**。MCP 只管通信不管鉴权，Server 拿什么凭证、能改删什么，责任在你。写操作先做确认机制，或先提供只读降级版本。
5. **传输方式过时**。老教程里的 SSE 传输已被 Streamable HTTP 取代，远程部署时注意协议版本（2025-03-26 起），别按旧文档配。

## 可复用建议

- 新 Server 先只提供只读工具，稳定运行一段时间再上写操作。
- 一个 Server 对应一个业务域（数据库一个、CI 一个），别做全家桶。
- 把工具命名和描述当产品文案来写，这部分投入的收益远大于反复调 prompt。
- 服务端自己记完整请求日志，出问题时宿主侧往往只剩一句笼统报错。

## 总结

MCP 没有发明新魔法，它做的是工程上最值钱的事：把重复造的轮子收拢成一个协议。对 Agent 开发者来说，价值在于工具生态第一次有了通用接口——你写的 Server 不再绑定某个宿主，你用的 Host 也不必为每个数据源定制集成。入门成本很低，一个 stdio Server 加 Inspector 就能跑通；真正的功课在权限边界和工具设计上，这两点决定了你的 Agent 是可靠还是"抽风"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/d743d6d092ebe6ba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/23f74565056a1d19.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-03/140ce877f3dffd96.png)

