---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42292
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景：Agent 接工具的"老办法"不好使了

做 Agent 自动化的同行大概率经历过这个阶段：想让模型读数据库，写一个 function calling 适配器；想让它操作 GitHub，再写一个；换个框架，这堆适配器全部重写。工具越多、宿主越多，工作量按 M×N 增长，且每家的参数格式、鉴权方式、错误处理都不一样，基本没有复用可言。

MCP（Model Context Protocol）就是冲着这个问题来的。2024 年底 Anthropic 将其开源，2025 年起被多家厂商的客户端跟进，现在基本成了 Agent 接外部工具和数据的事实标准。它的本质是一层接口约定：工具方实现一次 MCP Server，任何支持 MCP 的宿主（Agent 框架、IDE、OpenClaw 这类自动化平台）都能直接用。

## MCP 到底规定了什么

协议不复杂，核心是三件事：

1. **传输层**：本地场景用 stdio（宿主把 Server 当子进程拉起，走标准输入输出），远程场景用 Streamable HTTP。消息体是 JSON-RPC 2.0，抓包就能看懂。
2. **能力原语**：Server 可暴露三类东西——Tools（模型可调用的动作）、Resources（可读取的数据，如文件、API 返回）、Prompts（预置提示词模板）。绝大多数自动化场景用的都是 Tools。
3. **发现机制**：宿主连上 Server 后先调 `tools/list` 拿到工具清单和 JSON Schema，再按需 `tools/call`。Agent 不需要硬编码任何工具信息。

对应到实践里，M×N 的问题变成了 M+N：写 5 个 Server、接 3 个宿主，以前要维护 15 套适配器，现在 8 份实现就够。

## 从零接入的最小路径

以给自家 Agent 加"读本地文件 + 查 Git 日志"能力为例：

1. 选 Server。先别自己写，官方和社区已有 filesystem、git 等现成实现，直接安装即可。
2. 在宿主配置里声明 Server：命令、参数、环境变量，本地 Server 基本就是几行 JSON。
3. 启动后先列工具，确认宿主真的拿到了 `read_file`、`git_log` 及其参数 Schema。
4. 用自然语言触发一次真实调用，观察入参和返回是否符合预期。
5. 链路确认可用后，再考虑把内部 API 封装成自己的 Server（官方有 Python/TypeScript SDK，一两百行就能跑起来）。

## 踩坑点（都是付过学费的）

- **工具描述比代码重要**。模型选不选这个工具、传参对不对，几乎全看 description 和 Schema 写得清不清楚。含糊的描述会导致模型乱调或干脆不用。
- **工具数量失控**。挂 40 个工具上去，上下文膨胀、选择准确率下降是必然的。按需启用，一个 Server 只管一个领域。
- **权限别开自动放行**。MCP Server 是以你的身份在跑的，filesystem Server 指到根目录就是全盘可读写；工具返回内容里也可能藏注入指令，高危操作务必留人工确认。
- **stdio 的环境问题**。Windows 下的 npx shim、相对路径、缺失的环境变量，是"本机好好的，换台机器就挂"的三大来源。配置里一律写绝对路径。
- **别用已废弃的传输**。早期文档里的 HTTP+SSE 方案已被 Streamable HTTP 取代，新旧客户端混用会出现握手失败的诡异现象，先确认双方协议版本。

## 可复用的建议

- 先用现成 Server 验证链路，再写自己的；如果只有一个宿主要接一两个工具，原生 function calling 可能反而更简单。
- 工具设计尽量幂等、返回紧凑：分页查询远好于一次吐 10 万 token 的结果。
- 给工具调用留日志：入参、返回、耗时。出问题时这是唯一的排障线索。
- 把 Server 当独立服务对待：有版本号、有 README、有最小测试。

## 总结

MCP 没有解决 Agent 的编排、记忆、评测问题，它只解决一件事：**工具与数据的接入标准化**。但这件事恰恰是之前最脏最重复的活。如果你的场景是多宿主、多数据源，MCP 几乎是必选项；如果只是单框架内接一两个工具，也不必为了协议而协议。先把一个现成 Server 跑通，再决定要不要深入，是最省力的入门路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/b26abfd3b66d400d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/d53904deb71bacaa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/cbb3539d09ebb6b1.png)

