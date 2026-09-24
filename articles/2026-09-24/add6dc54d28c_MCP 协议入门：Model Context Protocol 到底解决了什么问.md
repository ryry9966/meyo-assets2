---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38761
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

过去一年做 Agent 自动化，最耗时间的往往不是提示词调优，而是“接线”：让模型能读本地文件、查数据库、调内部 API。每个宿主环境——IDE 插件、桌面客户端、自研 Agent 框架——都各自实现一套工具调用格式，互不兼容。同一个 GitHub 工具，在 A 框架写一遍适配器，换到 B 客户端还得再写一遍。

## 问题：M×N 集成困境

把问题抽象一下：M 个 AI 应用要接 N 个数据源/工具，最坏情况要维护 M×N 份胶水代码。工具方被迫为每个客户端出 SDK，客户端被迫为每个工具写适配，能力无法沉淀复用，权限管控也没有统一入口。

MCP（Model Context Protocol）做的事就一件：把 M×N 压缩成 M+N。应用侧实现一次 Client，工具侧实现一次 Server，中间用标准协议对话。

## 核心概念与上手步骤

MCP 的架构分三个角色：

- **Host**：模型运行的环境，比如桌面客户端、IDE、你自己的 Agent
- **Client**：Host 内部组件，负责和一个 Server 保持连接
- **Server**：暴露能力的服务，通常是一个独立进程

Server 通过三类原语暴露能力：**Tools**（模型可调用的动作）、**Resources**（可读取的上下文数据）、**Prompts**（预置提示模板）。消息层是 JSON-RPC 2.0，本地进程走 stdio 传输，远程服务走 Streamable HTTP。

上手四步：

1. 用官方 SDK（TypeScript/Python）起一个 Server 项目
2. 定义工具的输入 schema 和描述
3. 注册 handler，本地跑通
4. 在 Host 的配置文件里挂载 Server 命令，重启验证调用链

一个最小示例（Python）：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("fs-tools")

@mcp.tool()
def list_dir(path: str) -> list[str]:
    """列出指定目录下的文件名"""
    return os.listdir(path)

mcp.run(transport="stdio")
```

## 踩坑点

1. **stdio 传输下别往 stdout 打日志**。stdout 是协议通道，print 一句调试信息整条消息流就废了，日志一律走 stderr。
2. **工具描述是给模型看的接口文档**。写得含糊，模型要么不调、要么传错参数。`list_dir` 比 `handle_path` 好用得多。
3. **工具数量会撑爆上下文**。挂 20 个 Server 每个十几个工具，光 schema 就占掉大半窗口，按需启用。
4. **能力即风险**。Server 给了模型什么，模型就能做什么：路径加白名单、默认只读、密钥走环境变量，别图省事。
5. **提示注入**。外部数据经 Resources 进入上下文时可能夹带恶意指令，敏感操作务必要求人工确认。

## 可复用建议

- **先判断值不值**：单一宿主 + 一两个工具，直接写 function calling 更简单；只有需要跨多个宿主复用时，MCP 的标准化才划算。
- **一个 Server 只管一个领域**，细粒度组合优于大而全。
- **命名加前缀避免冲突**，schema 字段少而明确。
- **Server 侧记录调用日志**，出问题时能回放排障。

## 总结

MCP 没有任何魔法，它解决的就是集成成本问题：用一层标准协议，把“模型怎么用工具”和“工具怎么实现”解耦。理解了这一点，就知道什么时候该用它、什么时候一个普通函数就够了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/08e00449c65e119a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/065f4703b2429073.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/8e64514970557628.png)

