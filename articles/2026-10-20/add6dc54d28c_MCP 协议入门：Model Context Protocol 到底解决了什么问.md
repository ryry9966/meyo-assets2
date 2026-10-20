---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42102
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

过去一年做 Agent 自动化的人，大概率都被同一件事折磨过：接工具。让 Agent 查数据库、读本地文件、调内部 API、操作浏览器——每接一个，就要写一段胶水代码、定义一套 tool schema、调一遍参数格式。更糟的是换个框架或宿主，这些集成基本要重写一遍。

MCP（Model Context Protocol）就是冲着这个来的。它是 2024 年底开源的一项协议，一句话概括：给 LLM 应用和外部工具/数据源之间定一个标准接口。

## 它到底解决什么问题

核心是 **M×N → M+N** 的变换：

- 没有 MCP：M 个 Agent 宿主对接 N 个工具，最坏要写 M×N 份集成代码；
- 有了 MCP：每个工具暴露一个标准 MCP Server，每个宿主实现一个 MCP Client，两边按协议说话，集成成本降成 M+N。

注意，"tool calling" 本身并不解决这个问题。各家模型都有 function calling，但工具怎么被发现、怎么描述、鉴权怎么做、进程怎么拉起，全是各写各的。MCP 把这层“管道”标准化了，具体拆成三个原语：

- **Tools**：模型主动调用的动作（执行类）
- **Resources**：应用侧控制的只读上下文（读取类）
- **Prompts**：用户可触发的预设模板

外加两个实际很关键的机制：**运行时动态发现**（Client 连上后 list tools，无需硬编码）和**统一传输层**（本地 stdio、远程 Streamable HTTP）。

## 动手跑通的最小路径

以本地 stdio 为例，五步：

1. **选宿主**：任意支持 MCP 的客户端（Claude Desktop、各类 Agent 框架，OpenClaw 这类运行时也在跟进）。
2. **挂现成 Server**：先别自己写，用官方 filesystem 或 sqlite Server，改配置指定启动命令即可。
3. **验证发现**：重启宿主，确认工具列表出现，发一句“列出当前目录文件”验证调用链路。
4. **用 Inspector 调试**：官方 inspector 能单独拉起一个 Server，可视化查看工具定义与返回，比在宿主里盲猜快得多。
5. **写自己的 Server**：用 Python/TS SDK，十几行就能把一个内部 HTTP API 包成 MCP 工具。

## 踩坑点

都是实际会撞上的：

1. **描述质量决定一切。** 模型选工具靠 name + description，`process_data` 这种名字等于没写。描述里写清楚什么时候该用、什么时候不该用。
2. **工具数量失控。** 一次挂 40 个工具，上下文先爆，选择准确率也跟着掉。按需启用，别图全。
3. **stdio 的环境继承问题。** Server 是宿主拉起的子进程，venv 路径、相对路径、环境变量经常对不上，报错还藏在宿主日志里。启动命令一律写绝对路径。
4. **安全边界。** Server 返回内容会进模型上下文，等于一条提示词注入通道。文件系统、密钥类 Server 只给最小权限，别在不受信环境挂高权限 Server。
5. **Tool 和 Resource 别混用。** 改状态的操作走 Tool，只读上下文走 Resource，混着来模型会用错。
6. **协议版本在动。** 早期远程传输是 SSE，后来换成了 Streamable HTTP，照抄旧教程会遇到连不上的情况，先核对 spec 版本。

## 可复用建议

- **一个领域一个 Server**，单 Server 工具数控制在个位数；
- **内部系统包 MCP Server 做一次，所有 Agent 复用**——这是回报最高的投入；
- **每次工具调用记日志**（参数 + 返回），出问题时有据可查；
- **工具粒度宁粗勿细**：一个“创建工单”好过“校验参数→查重→提交”三件套，多步编排是 Agent 的活，不是工具的活。

## 总结

MCP 解决的是**接入层的标准化**：工具怎么被发现、怎么被描述、怎么被调用。它不解决工具设计本身的好坏——描述写得烂、粒度切得碎，协议再标准模型也不会用。把它当水管，别当银弹：先跑通一个官方 Server 建立体感，再回头整理自己那堆一次性集成脚本，大部分都能收敛掉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/0d497bbea7cd556a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/02541ed5f5ff9a0d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/7f305c9ce9dedd9b.png)

