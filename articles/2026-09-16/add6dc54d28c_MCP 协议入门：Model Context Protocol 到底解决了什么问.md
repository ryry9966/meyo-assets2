---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37748
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

做大模型应用绕不开一件事：模型只会生成文本，真正干活要靠外部工具——查数据库、调内部 API、读写文件。Function calling 解决了"模型怎么表达想调工具"，但没解决"应用怎么接入工具"。这正是 MCP 补的位置。

## 问题：M×N 集成困境

MCP 出现之前是典型的 M×N 问题：M 个 AI 应用（桌面端、IDE 插件、自研 Agent）× N 个工具（GitHub、数据库、内部系统），每对组合都要单独写适配。更麻烦的是各家定义工具的方式不统一——schema 格式、鉴权、传输方式各搞一套。工具开发者要为每个应用维护一份适配代码；应用开发者每接一个新工具就要写一套胶水层。系统一多，维护成本线性爆炸。

## MCP 的做法

MCP（Model Context Protocol）是 Anthropic 于 2024 年 11 月开源的协议，它做的事很克制：**定义 Host 与 Server 之间的标准通信方式**。

核心概念三个：

- **Tools**：模型可以主动调用的能力（如"查询订单"）
- **Resources**：应用可读取的上下文数据（如文件、配置）
- **Prompts**：预置的提示词模板

架构上，消息走 JSON-RPC 2.0，传输层支持 stdio（本地子进程）或 Streamable HTTP（远程服务）。Host 启动时连接 Server，通过 initialize 握手协商协议版本与能力。

一句话概括价值：**M×N 变成 M+N**。工具方只需实现一次 MCP Server，任何支持 MCP 的宿主都能直接用。

## 从零跑通一个最小 Server

1. 环境：Python 3.10+，安装 `pip install "mcp[cli]"`
2. 写一个最小 server：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回状态和物流信息。"""
    return lookup(order_id)

mcp.run()  # 默认 stdio 传输
```

3. 在 Host 的配置文件里注册该 server（如 Claude Desktop 的 `claude_desktop_config.json`）
4. 先用 `mcp dev server.py` 启动 Inspector，确认工具列表加载正常、手动调用返回正确，再接入宿主
5. 跑通后逐步加 Resources、鉴权和错误处理

## 踩坑记录

- **stdio 模式严禁往 stdout 打日志**。任何 `print` 都会污染 JSON-RPC 消息流，宿主侧报错非常隐晦。日志一律走 stderr。
- **工具描述就是写给模型的 prompt**。描述含糊，模型就选错工具或乱填参数；参数格式也要在 description 里写清楚。
- **schema 别贪复杂**。嵌套过深的参数结构，较弱的模型填不对，尽量扁平、必填项少。
- **协议版本不匹配**会导致握手失败，有些宿主只报一句 initialize failed。排查时先升级 SDK。
- **Windows 下 stdio 有编码坑**，子进程默认 GBK，中文返回乱码，需显式指定 UTF-8。
- **安全不能省**：接入第三方 MCP Server 等于运行别人写的代码，工具描述里可能藏注入（tool poisoning）。生产环境建议白名单 + 最小权限。

## 可复用建议

- 一个领域一个 Server，工具保持小而正交，宁可多个小工具，不要一个大而全
- 描述按 prompt 标准写，写完先在 Inspector 里人肉测一遍再上线
- 长耗时任务做进度通知，或拆成"提交 + 查询状态"两个工具
- 能无状态就无状态，会话状态尽量放 Host 侧管理

## 总结

MCP 不是什么智能增强，它就是一层管道标准化：把工具接入成本从 M×N 压到 M+N。单个应用接单个工具时感知不强，但只要你有两个以上宿主或三个以上工具，收益立刻显现。建议路径：先用 Inspector 跑通 demo，再挑内部最稳定的一个系统包成 Server，稳定运行两周后再考虑铺开。协议本身很薄，真正的功夫在工具设计和权限边界上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/76bf2305386e3ced.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/bfdd83b58ebec414.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/9299dfc67603f09b.png)

