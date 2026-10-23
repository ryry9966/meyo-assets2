---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42403
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

过去一年做 Agent 的同学几乎绕不开一个词：MCP（Model Context Protocol）。它由 Anthropic 在 2024 年底提出并开源，之后被主流 IDE、聊天客户端和 Agent 框架陆续支持。社区里“给 Agent 接 XX 能力”的需求，第一反应也变成了“有没有现成的 MCP server”。但在动手之前值得先想清楚：它解决的到底是什么层面的痛点。

## 问题：M×N 的胶水代码

LLM 本身只会生成文本。要让 Agent 操作文件、查数据库、调内部 API，必须由宿主程序提供工具调用通道。MCP 之前，这件事是手工作坊式的：

- 每个客户端（你的 Bot、IDE 插件、自动化脚本）都要为每个工具单独写一遍对接：定义 schema、处理鉴权、解析返回、兜底错误；
- M 个客户端 × N 个工具，就是 M×N 份互不相通的胶水代码；
- 你给自己 Bot 写的工具封装，换一个框架基本作废；别人写的工具你也接不进来。

本质不是“缺工具”，而是缺一个统一的接入标准。

## MCP 的做法：一次实现，处处可插

MCP 把 M×N 收敛成 M+N：客户端各实现一次协议，工具侧各暴露一次服务。工程视角三句话：

1. **架构是 client/server，消息走 JSON-RPC 2.0**。宿主程序（比如 OpenClaw）内置 client，工具侧跑一个 server，通过 stdio（本地子进程）或 Streamable HTTP（远程服务）通信。
2. **server 暴露三类能力**：Tools（模型可调用的动作）、Resources（可读取的上下文数据）、Prompts（提示词模板）。日常自动化 90% 的场景只用 Tools。
3. **接入是配置，不是编码**。选一个现成 server（filesystem、playwright、sqlite 等官方与社区维护了一批），在客户端配置里声明启动命令即可：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "~/project"]
    }
  }
}
```

（字段名以你所用客户端文档为准，思路一致。）重启会话后，client 拉取工具列表，把 schema 注入模型上下文，模型按需发起调用，client 转发给 server 执行并回填结果。

## 踩坑点

- **工具越多不一定越强**。几十个工具的 schema 全塞进上下文，token 消耗大，模型路由也容易选错。按场景只挂当前任务需要的 server。
- **工具描述就是给模型看的 API 文档**。写得含糊，模型就乱调或干脆不调。自己写 server 时，description 用动词开头，参数写清单位与约束。
- **stdio server 静默失败**。npx/uvx 不在 PATH、Windows 命令名不同、依赖首装超时，表现往往是“工具列表为空”。先手动跑一遍启动命令看 stderr。
- **安全边界要想清楚**。挂载一个 server 等于把一段任意代码以你的权限跑在本机。第三方 server 先审来源，最小授权（如 filesystem 只给子目录）。
- **Resource ≠ Tool**。Resources 不会自动进入模型上下文，是否读取由宿主决定；想让模型“主动能调”的东西，做成 Tool。
- **远程模式注意鉴权与超时**。OAuth 流程、长任务超时、结果过大撑爆上下文，都是实际会撞上的坑。

## 可复用建议

1. 从 2~3 个高频 server 起步（文件系统 + 一个业务系统），链路验证通了再扩展。
2. 内部 API 不要整库暴露，写一个薄封装 server，只暴露粒度合适的动作，鉴权收敛在一处。
3. 把 MCP 配置当代码管理：进版本库、写注释、锁版本，换机十分钟恢复。
4. 给工具调用留日志。排查“Agent 为什么没调这个工具”时，先确认 client 是否真的把工具注入了上下文。

## 总结

MCP 不是让模型变强的魔法，它只是一根标准化的水管：把“每个客户端对每个工具写一遍胶水”收敛成“各实现一次协议”。对个人开发者，价值在于现成生态直接复用；对团队，是把工具能力从某个框架的私有插件体系里解放出来。判断标准很简单：只服务一个客户端，直接写函数调用更省事；一旦工具要在多个宿主间复用，MCP 的成本优势立刻显现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/44c63d13567074a4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/8a74581e31bcffc4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/e1e112834adb87bb.png)

